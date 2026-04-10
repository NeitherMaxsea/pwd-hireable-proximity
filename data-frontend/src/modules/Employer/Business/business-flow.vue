<script setup>
import '@fontsource/inter/400.css'
import '@fontsource/inter/500.css'
import '@fontsource/inter/600.css'
import '@fontsource/inter/700.css'
import { computed, markRaw, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { useRouter } from 'vue-router'
import BNavbar from '@/modules/Employer/Business/business_Navbar.vue'
import BSidebar from '@/modules/Employer/Business/business_Sidebar.vue'
import BUserNavbar from '@/modules/Employer/Business/business_user_navbar.vue'
import BUserSidebar from '@/modules/Employer/Business/business_user_sidebar.vue'
import BusinessDashboardStage from '@/modules/Employer/Business/business-dashboard.vue'
import BusinessJobPostingStage from '@/modules/Employer/Business/business-job-posting.vue'
import BusinessApplicantManagementStage from '@/modules/Employer/Business/business-applicant-management.vue'
import BusinessAssessmentStage from '@/modules/Employer/Business/business-assessment-stage-live.vue'
import BusinessContractSigningStage from '@/modules/Employer/Business/business-contract-signing-stage.vue'
import BusinessIssueOfferStage from '@/modules/Employer/Business/business-issue-offer-stage.vue'
import BusinessInterviewStage from '@/modules/Employer/Business/business-interview-stage.vue'
import BusinessSubscriptionStage from '@/modules/Employer/Business/business-subscription-stage.vue'
import BusinessTeamMembersStage from '@/modules/Employer/Business/business-team-members-stage.vue'
import BusinessTrainingStage from '@/modules/Employer/Business/business-training-stage.vue'
import { createDashboardState } from '@/modules/Employer/Business/business_dashboard_bindings'
import pwdLogo from '@/assets/logo-pwd.png'
import './business_shell.css'
import {
  clearAuthSession,
  getStoredAuthUser,
  subscribeToBusinessMemberEmployers,
  subscribeToBusinessWorkspaceUsers,
  updateEmployerAdminDetails,
  uploadEmployerBusinessAvatar,
} from '@/lib/auth'
import { subscribeToBusinessJobApplications } from '@/lib/apply_jobs'
import { subscribeToWorkspaceJobs } from '@/lib/jobs'

const router = useRouter()
const ownerSidebarComponent = markRaw(BSidebar)
const employeeSidebarComponent = markRaw(BUserSidebar)
const ownerNavbarComponent = markRaw(BNavbar)
const employeeNavbarComponent = markRaw(BUserNavbar)

const text = (value = '') => String(value ?? '').trim()
const emailText = (value = '') => text(value).toLowerCase()
const buildInitials = (value = '') =>
  text(value)
    .split(/\s+/)
    .filter(Boolean)
    .slice(0, 2)
    .map((part) => part.charAt(0).toUpperCase())
    .join('') || 'BW'

const isRenderableAvatar = (value = '') => /^(https?:|data:|blob:|\/)/i.test(text(value))

const storedUser = ref(getStoredAuthUser())
const activeSection = ref('dashboard')
const activeSidebarGroup = ref('dashboard')
const expandedSidebarGroups = ref([])
const isProfileMenuOpen = ref(false)
const isNotificationMenuOpen = ref(false)
const isLogoutSubmitting = ref(false)
const isLogoutConfirmOpen = ref(false)
const businessNavbarNotifications = ref([
  {
    id: 'business-workspace-ready',
    title: 'Business workspace ready',
    message: 'Sidebar and navigation are active. Backend-heavy content stays disabled while debugging.',
    createdAt: new Date().toISOString(),
    targetSection: 'dashboard',
    read: false,
    tone: 'success',
  },
])
const businessJobPosts = ref([])
const businessJobApplications = ref([])
const workspaceUserDirectory = ref([])
const employeeDirectory = ref([])
const profileAvatarInputRef = ref(null)
const isProfileAvatarLoading = ref(false)
const profileStatusMessage = ref('')
const profileStatusTone = ref('success')
const profileForm = ref({
  companyName: '',
  email: '',
  category: '',
  location: '',
  contactPerson: '',
  avatar: '',
})

const storedUserId = computed(() => text(storedUser.value?.id || storedUser.value?.uid))
const workspaceOwnerId = computed(() => text(storedUser.value?.workspace_owner_id || storedUser.value?.workspaceOwnerId))
const workspaceOwnerEmail = computed(() =>
  emailText(storedUser.value?.workspace_owner_email || storedUser.value?.workspaceOwnerEmail),
)
const storedUserEmail = computed(() =>
  emailText(storedUser.value?.email || storedUser.value?.gmail || storedUser.value?.business_contact_email),
)
const hasWorkspaceRole = computed(() => text(storedUser.value?.roleId || storedUser.value?.permissionRoleId) !== '')
const businessWorkspaceOwnerId = computed(() =>
  text(
    storedUser.value?.workspace_owner_id
    || storedUser.value?.workspaceOwnerId
    || storedUser.value?.id
    || storedUser.value?.uid,
  ),
)
const businessWorkspaceOwnerEmailForSync = computed(() =>
  emailText(
    storedUser.value?.workspace_owner_email
    || storedUser.value?.workspaceOwnerEmail
    || storedUser.value?.email
    || storedUser.value?.gmail
    || storedUser.value?.business_contact_email,
  ),
)

const isBusinessEmployeeWorkspaceMode = computed(() => {
  if (storedUser.value?.workspace_member === true) return true
  if (workspaceOwnerId.value && storedUserId.value && workspaceOwnerId.value !== storedUserId.value) return true
  if (!workspaceOwnerId.value && workspaceOwnerEmail.value && storedUserEmail.value && workspaceOwnerEmail.value !== storedUserEmail.value) {
    return true
  }

  return hasWorkspaceRole.value && workspaceOwnerId.value && workspaceOwnerId.value !== storedUserId.value
})

const businessWorkspaceOwnerName = computed(() =>
  text(
    storedUser.value?.workspace_owner_name
    || storedUser.value?.workspaceOwnerName
    || storedUser.value?.company_name
    || storedUser.value?.business_name
    || storedUser.value?.name,
  ) || 'Business Workspace',
)

const businessSidebarBrandName = computed(() =>
  isBusinessEmployeeWorkspaceMode.value
    ? businessWorkspaceOwnerName.value
    : text(storedUser.value?.company_name || storedUser.value?.business_name || storedUser.value?.name) || 'Business Workspace',
)

const businessSidebarBrandSubtitle = computed(() =>
  isBusinessEmployeeWorkspaceMode.value ? 'Employee Workspace' : 'Business Workspace',
)

const loggedInBusinessUserName = computed(() =>
  text(storedUser.value?.name || storedUser.value?.full_name || storedUser.value?.company_name) || 'Business User',
)

const loggedInBusinessUserEmail = computed(() =>
  emailText(storedUser.value?.email || storedUser.value?.gmail || storedUser.value?.business_contact_email),
)

const loggedInBusinessUserInitials = computed(() => buildInitials(loggedInBusinessUserName.value))
const loggedInBusinessUserAvatar = computed(() => {
  const avatar = text(storedUser.value?.avatar || storedUser.value?.profile_avatar || storedUser.value?.photoURL || storedUser.value?.photo_url)
  return isRenderableAvatar(avatar) ? avatar : ''
})

const loggedInBusinessUserRoleLabel = computed(() => {
  if (isBusinessEmployeeWorkspaceMode.value) {
    return text(storedUser.value?.role_name || storedUser.value?.roleName || storedUser.value?.position || storedUser.value?.role) || 'Workspace User'
  }

  return 'Business Owner'
})
const businessProfileName = computed(() =>
  text(profileForm.value.companyName || businessSidebarBrandName.value) || 'Business Workspace',
)
const businessProfileEmail = computed(() =>
  emailText(profileForm.value.email || loggedInBusinessUserEmail.value),
)
const businessProfileCategory = computed(() =>
  text(profileForm.value.category || storedUser.value?.company_category) || 'Not set',
)
const businessProfileLocation = computed(() =>
  text(profileForm.value.location || storedUser.value?.company_location) || 'Not set',
)
const businessProfileContactPerson = computed(() =>
  text(profileForm.value.contactPerson || storedUser.value?.name || storedUser.value?.full_name) || 'Not set',
)
const businessProfileInitials = computed(() => buildInitials(businessProfileName.value))

const businessNavbarSettingsItems = computed(() => {
  const items = [
    { label: 'Profile', icon: 'bi bi-person-circle' },
  ]

  if (!isBusinessEmployeeWorkspaceMode.value) {
    items.unshift({ label: 'Subscriptions', icon: 'bi bi-stars' })
  }

  return items
})

const sectionMeta = {
  dashboard: {
    id: 'dashboard',
    label: 'Dashboard',
    title: 'Business Dashboard',
    description: 'Sidebar, navbar, and login shell are active again while backend-heavy content stays paused.',
    eyebrow: 'Business Workspace',
  },
  'job-posting': {
    id: 'job-posting',
    label: 'Job Posting',
    title: 'Job Posting',
    description: 'Create, edit, and manage live business job posts from one dedicated workspace screen.',
    eyebrow: 'Job Posting',
  },
  'applicant-management': {
    id: 'applicant-management',
    label: 'Applicant Management',
    title: 'Applicant Management',
    description: 'Applicant tools are temporarily hidden, but the workspace route and sidebar stay available.',
    eyebrow: 'Recruitment',
  },
  'issue-offer': {
    id: 'issue-offer',
    label: 'Issue Offer',
    title: 'Issue Offer',
    description: 'Offer creation is currently in placeholder mode while the Business backend stays disabled.',
    eyebrow: 'Recruitment',
  },
  'contract-signing': {
    id: 'contract-signing',
    label: 'Contract Signing',
    title: 'Contract Signing',
    description: 'Send contract files, receive signed copies back from applicants, and monitor the exchange in real time.',
    eyebrow: 'Recruitment',
  },
  'assessment-management': {
    id: 'assessment-management',
    label: 'Create Assessment',
    title: 'Create Assessment',
    description: 'Build and edit screening templates inside the restored assessment workspace.',
    eyebrow: 'Assessment',
  },
  'assessment-assignment': {
    id: 'assessment-assignment',
    label: 'Assign Assessment',
    title: 'Assign Assessment',
    description: 'Assign assessment templates to applicants and keep assignment status visible in the same workspace.',
    eyebrow: 'Assessment',
  },
  'applicant-score': {
    id: 'applicant-score',
    label: 'Applicant Score',
    title: 'Applicant Score',
    description: 'Track applicant assessment scores inside the restored assessment workspace.',
    eyebrow: 'Assessment',
  },
  'interview-scheduling': {
    id: 'interview-scheduling',
    label: 'Interview Scheduling',
    title: 'Interview Scheduling',
    description: 'Create live interview schedules and send them directly to the applicant interview page.',
    eyebrow: 'Recruitment',
  },
  'interview-status': {
    id: 'interview-status',
    label: 'Interview Status',
    title: 'Interview Status',
    description: 'Review applicant interview responses, reschedule requests, and live interview outcomes in one table.',
    eyebrow: 'Recruitment',
  },
  'training-templates': {
    id: 'training-templates',
    label: 'Training Templates',
    title: 'Training Templates',
    description: 'Build training templates and review progress with the restored business design.',
    eyebrow: 'Training',
  },
  'user-overview': {
    id: 'user-overview',
    label: 'Team Manager',
    title: 'Team Manager',
    description: 'Review workspace users, employee records, and team access in the restored frontend view.',
    eyebrow: 'Workspace',
  },
  'create-user': {
    id: 'create-user',
    label: 'Create User',
    title: 'Create User',
    description: 'Prepare workspace account details with the restored create-user design.',
    eyebrow: 'Team Members',
  },
  'add-employee': {
    id: 'add-employee',
    label: 'Employee Directory',
    title: 'Employee Directory',
    description: 'Browse the restored employee directory layout inside the Business workspace.',
    eyebrow: 'Team Members',
  },
  permissions: {
    id: 'permissions',
    label: 'Permissions',
    title: 'Permissions',
    description: 'Review permission roles and access levels in the restored frontend permissions screen.',
    eyebrow: 'Team Members',
  },
  profile: {
    id: 'profile',
    label: 'Profile',
    title: 'Business Profile',
    description: 'Profile editing is paused for now, but the account shell remains visible after login.',
    eyebrow: 'Workspace',
  },
  subscriptions: {
    id: 'subscriptions',
    label: 'Subscriptions',
    title: 'Subscriptions',
    description: 'Subscription controls are temporarily replaced with a lightweight placeholder.',
    eyebrow: 'Workspace',
  },
}

const ownerSidebarGroups = [
  {
    id: 'dashboard',
    label: 'Dashboard',
    icon: 'bi bi-grid-1x2-fill',
    items: [{ id: 'dashboard', label: 'Dashboard' }],
  },
  {
    id: 'job-posting-group',
    label: 'Job Posting',
    icon: 'bi bi-briefcase-fill',
    items: [
      { id: 'job-posting', label: 'Job Posting' },
      { id: 'applicant-management', label: 'Applicant Management' },
      { id: 'issue-offer', label: 'Issue Offer' },
      { id: 'contract-signing', label: 'Contract Signing' },
    ],
  },
  {
    id: 'assessment',
    label: 'Assessment Management',
    icon: 'bi bi-ui-checks-grid',
    items: [
      { id: 'assessment-management', label: 'Create Assessment' },
      { id: 'assessment-assignment', label: 'Assign Assessment' },
      { id: 'applicant-score', label: 'Applicant Score' },
    ],
  },
  {
    id: 'interview',
    label: 'Interview Management',
    icon: 'bi bi-calendar-event-fill',
    items: [
      { id: 'interview-scheduling', label: 'Interview Scheduling' },
      { id: 'interview-status', label: 'Interview Status' },
    ],
  },
  {
    id: 'training',
    label: 'Training Management',
    icon: 'bi bi-journal-richtext',
    items: [
      { id: 'training-templates', label: 'Training Templates' },
    ],
  },
  {
    id: 'employees',
    label: 'Team Members',
    icon: 'bi bi-people-fill',
    items: [
      { id: 'user-overview', label: 'Team Overview' },
      { id: 'create-user', label: 'Create User' },
      { id: 'add-employee', label: 'Employee Directory' },
      { id: 'permissions', label: 'Permissions' },
    ],
  },
]

const employeeSidebarGroups = [
  {
    id: 'dashboard',
    label: 'Dashboard',
    icon: 'bi bi-grid-1x2-fill',
    items: [{ id: 'dashboard', label: 'Dashboard' }],
  },
  {
    id: 'job-posting-group',
    label: 'Assigned Tools',
    icon: 'bi bi-person-workspace',
    items: [
      { id: 'job-posting', label: 'Job Posting' },
      { id: 'applicant-management', label: 'Applicant Management' },
      { id: 'user-overview', label: 'Team Manager' },
    ],
  },
  {
    id: 'workspace',
    label: 'Account',
    icon: 'bi bi-person-badge-fill',
    items: [
      { id: 'profile', label: 'Profile' },
    ],
  },
]

const sidebarGroups = computed(() =>
  isBusinessEmployeeWorkspaceMode.value ? employeeSidebarGroups : ownerSidebarGroups,
)
const activeSidebarComponent = computed(() =>
  isBusinessEmployeeWorkspaceMode.value ? employeeSidebarComponent : ownerSidebarComponent,
)
const activeNavbarComponent = computed(() =>
  isBusinessEmployeeWorkspaceMode.value ? employeeNavbarComponent : ownerNavbarComponent,
)

const currentSection = computed(() => sectionMeta[activeSection.value] || sectionMeta.dashboard)
const businessNavbarBreadcrumbParent = computed(() =>
  isBusinessEmployeeWorkspaceMode.value ? 'Employee Workspace' : 'Business Workspace',
)
const businessSidebarSecondarySectionLabel = computed(() =>
  isBusinessEmployeeWorkspaceMode.value ? 'Employee Access' : 'Quick Access',
)
const showBusinessSubscriptionShortcut = computed(() => !isBusinessEmployeeWorkspaceMode.value)
const showBusinessProfileShortcut = computed(() => true)
const unreadBusinessNotificationCount = computed(() =>
  businessNavbarNotifications.value.filter((item) => item.read !== true).length,
)
const hasDashboardSidebarLink = computed(() =>
  sidebarGroups.value.some((group) =>
    text(group.id) === 'dashboard'
    && Array.isArray(group.items)
    && group.items.some((item) => text(item.id) === 'dashboard'),
  ),
)
const sidebarRenderGroups = computed(() => {
  const currentActiveSection = activeSection.value
  const currentActiveGroup = activeSidebarGroup.value
  const isDashboardActive = currentActiveSection === 'dashboard'
  const expandedGroupIds = new Set(expandedSidebarGroups.value.map((entry) => text(entry)).filter(Boolean))

  return sidebarGroups.value
    .filter((group) => text(group.id) !== 'dashboard')
    .map((group) => {
      const items = Array.isArray(group.items) ? group.items : []
      const groupId = text(group.id)

      return {
        ...group,
        items,
        isExpanded: expandedGroupIds.has(groupId),
        isActive: !isDashboardActive
          && (currentActiveGroup === groupId || items.some((item) => text(item.id) === currentActiveSection)),
      }
    })
})
const visibleSectionCards = computed(() => {
  const seenSectionIds = new Set()
  const cards = []

  for (const group of sidebarGroups.value) {
    const items = Array.isArray(group.items) ? group.items : []
    for (const item of items) {
      const sectionId = text(item.id)
      if (!sectionId || seenSectionIds.has(sectionId)) continue

      seenSectionIds.add(sectionId)
      cards.push(item)
    }
  }

  return cards
})

const {
  dashboardSummaryCards,
  dashboardApplicantFeed,
  dashboardSyncOverview,
  dashboardBarSeries,
  dashboardDonutLegend,
  dashboardDonutTotal,
  dashboardDonutStyle,
  businessTrendChart,
} = createDashboardState({
  isBusinessEmployeeWorkspaceMode,
  businessJobPosts,
  businessJobApplications,
  workspaceUserDirectory,
  employeeDirectory,
})

let stopDashboardJobsSync = () => {}
let stopDashboardApplicationsSync = () => {}
let stopDashboardWorkspaceUsersSync = () => {}
let stopDashboardEmployeesSync = () => {}

const stopDashboardSyncSubscriptions = () => {
  stopDashboardJobsSync()
  stopDashboardApplicationsSync()
  stopDashboardWorkspaceUsersSync()
  stopDashboardEmployeesSync()
  stopDashboardJobsSync = () => {}
  stopDashboardApplicationsSync = () => {}
  stopDashboardWorkspaceUsersSync = () => {}
  stopDashboardEmployeesSync = () => {}
}

const syncProfileFormFromStoredUser = () => {
  profileForm.value = {
    companyName: text(
      storedUser.value?.company_name
      || storedUser.value?.business_name
      || storedUser.value?.workspace_owner_name
      || storedUser.value?.workspaceOwnerName
      || storedUser.value?.name,
    ),
    email: text(
      storedUser.value?.business_contact_email
      || storedUser.value?.email
      || storedUser.value?.gmail,
    ),
    category: text(storedUser.value?.company_category || storedUser.value?.category),
    location: text(storedUser.value?.company_location || storedUser.value?.location),
    contactPerson: text(storedUser.value?.name || storedUser.value?.full_name),
    avatar: text(
      storedUser.value?.business_avatar
      || storedUser.value?.business_avatar_path
      || storedUser.value?.avatar
      || storedUser.value?.profile_avatar
      || storedUser.value?.photoURL
      || storedUser.value?.photo_url,
    ),
  }
  isProfileAvatarLoading.value = false
}

const setProfileStatus = (message = '', tone = 'success') => {
  profileStatusMessage.value = text(message)
  profileStatusTone.value = text(tone) || 'success'
}

const openProfileAvatarPicker = () => {
  profileAvatarInputRef.value?.click()
}

const removeProfileAvatar = async () => {
  if (isProfileAvatarLoading.value || !storedUserId.value) return

  isProfileAvatarLoading.value = true
  try {
    await updateEmployerAdminDetails(storedUserId.value, {
      company_name: businessProfileName.value,
      name: businessProfileContactPerson.value,
      business_contact_email: businessProfileEmail.value,
      company_category: text(profileForm.value.category),
      company_location: text(profileForm.value.location),
      business_avatar: '',
      avatar: '',
      business_avatar_path: '',
      avatar_path: '',
    })

    profileForm.value = {
      ...profileForm.value,
      avatar: '',
    }
    storedUser.value = {
      ...(storedUser.value || {}),
      business_avatar: '',
      business_avatar_path: '',
      avatar: '',
      avatar_path: '',
      profile_avatar: '',
    }
    if (profileAvatarInputRef.value) profileAvatarInputRef.value.value = ''
    setProfileStatus('Profile photo removed.', 'success')
  } catch (error) {
    setProfileStatus(error instanceof Error ? error.message : 'Unable to remove profile photo right now.', 'error')
  } finally {
    isProfileAvatarLoading.value = false
  }
}

const handleProfileAvatarChange = (event) => {
  const file = event?.target?.files?.[0]
  if (!file || !storedUserId.value) return

  isProfileAvatarLoading.value = true

  uploadEmployerBusinessAvatar(storedUserId.value, file)
    .then((uploadedAvatar) => {
      const avatarUrl = text(uploadedAvatar?.url)
      const avatarPath = text(uploadedAvatar?.path)

      profileForm.value = {
        ...profileForm.value,
        avatar: avatarUrl,
      }

      storedUser.value = {
        ...(storedUser.value || {}),
        business_avatar: avatarUrl,
        business_avatar_path: avatarPath,
        avatar: avatarUrl,
        avatar_path: avatarPath,
        profile_avatar: avatarUrl,
      }
      setProfileStatus('Profile photo updated.', 'success')
    })
    .catch((error) => {
      setProfileStatus(error instanceof Error ? error.message : 'Unable to upload profile photo right now.', 'error')
    })
    .finally(() => {
      isProfileAvatarLoading.value = false
    })
}

const saveBusinessProfile = async () => {
  if (!storedUserId.value) return

  try {
    const updatedProfile = await updateEmployerAdminDetails(storedUserId.value, {
      company_name: businessProfileName.value,
      name: businessProfileContactPerson.value,
      business_contact_email: businessProfileEmail.value,
      company_category: text(profileForm.value.category),
      company_location: text(profileForm.value.location),
      business_avatar: text(profileForm.value.avatar),
      avatar: text(profileForm.value.avatar),
      business_avatar_path: text(
        storedUser.value?.business_avatar_path
        || storedUser.value?.avatar_path,
      ),
      avatar_path: text(
        storedUser.value?.avatar_path
        || storedUser.value?.business_avatar_path,
      ),
    })

    storedUser.value = {
      ...(storedUser.value || {}),
      ...updatedProfile,
      company_name: text(updatedProfile?.company_name || businessProfileName.value),
      business_name: text(updatedProfile?.company_name || businessProfileName.value),
      business_contact_email: text(updatedProfile?.business_contact_email || businessProfileEmail.value),
      company_category: text(updatedProfile?.company_category || profileForm.value.category),
      company_location: text(updatedProfile?.company_location || profileForm.value.location),
      name: text(updatedProfile?.name || businessProfileContactPerson.value),
      business_avatar: text(updatedProfile?.business_avatar || profileForm.value.avatar),
      avatar: text(updatedProfile?.avatar || updatedProfile?.business_avatar || profileForm.value.avatar),
    }
    setProfileStatus('Business profile updated successfully.', 'success')
  } catch (error) {
    setProfileStatus(error instanceof Error ? error.message : 'Unable to save the business profile right now.', 'error')
  }
}

const syncDashboardSubscriptions = () => {
  stopDashboardSyncSubscriptions()

  const workspaceOwnerId = businessWorkspaceOwnerId.value
  const ownerEmail = businessWorkspaceOwnerEmailForSync.value
  const actorId = storedUserId.value
  if (!workspaceOwnerId && !ownerEmail && !actorId) {
    businessJobPosts.value = []
    businessJobApplications.value = []
    workspaceUserDirectory.value = []
    employeeDirectory.value = []
    return
  }

  stopDashboardJobsSync = subscribeToWorkspaceJobs(
    {
      workspaceOwnerId,
      workspaceOwnerEmail: ownerEmail,
      actorId,
    },
    (records = []) => {
      businessJobPosts.value = Array.isArray(records) ? records : []
    },
    () => {
      businessJobPosts.value = []
    },
  )

  if (workspaceOwnerId) {
    stopDashboardApplicationsSync = subscribeToBusinessJobApplications(
      workspaceOwnerId,
      (records = []) => {
        businessJobApplications.value = Array.isArray(records) ? records : []
      },
      () => {
        businessJobApplications.value = []
      },
    )

    stopDashboardWorkspaceUsersSync = subscribeToBusinessWorkspaceUsers(
      workspaceOwnerId,
      (records = []) => {
        workspaceUserDirectory.value = Array.isArray(records) ? records : []
      },
      () => {
        workspaceUserDirectory.value = []
      },
    )

    stopDashboardEmployeesSync = subscribeToBusinessMemberEmployers(
      workspaceOwnerId,
      (records = []) => {
        employeeDirectory.value = Array.isArray(records) ? records : []
      },
      () => {
        employeeDirectory.value = []
      },
    )
  }
}

const getSidebarItemIcon = (sectionId = '') => {
  const iconMap = {
    dashboard: 'bi bi-grid-1x2-fill',
    'job-posting': 'bi bi-megaphone-fill',
    'applicant-management': 'bi bi-people-fill',
    'issue-offer': 'bi bi-envelope-paper-fill',
    'contract-signing': 'bi bi-pen-fill',
    'assessment-management': 'bi bi-ui-checks-grid',
    'assessment-assignment': 'bi bi-clipboard-check-fill',
    'applicant-score': 'bi bi-bar-chart-fill',
    'interview-scheduling': 'bi bi-calendar-event-fill',
    'interview-status': 'bi bi-table',
    'training-templates': 'bi bi-journal-text',
    'user-overview': 'bi bi-person-vcard-fill',
    'create-user': 'bi bi-person-plus-fill',
    'add-employee': 'bi bi-person-vcard-fill',
    permissions: 'bi bi-shield-lock-fill',
    profile: 'bi bi-person-circle',
    subscriptions: 'bi bi-stars',
  }

  return iconMap[text(sectionId)] || 'bi bi-circle-fill'
}

const findSidebarGroupBySection = (sectionId = '') =>
  sidebarGroups.value.find((group) =>
    Array.isArray(group.items) && group.items.some((item) => item.id === text(sectionId)),
  ) || null

const closeFloatingMenus = () => {
  if (!isProfileMenuOpen.value && !isNotificationMenuOpen.value) return
  isProfileMenuOpen.value = false
  isNotificationMenuOpen.value = false
}

const syncSection = (sectionId = 'dashboard') => {
  const normalizedSectionId = text(sectionId) || 'dashboard'
  activeSection.value = normalizedSectionId

  const targetGroup = findSidebarGroupBySection(normalizedSectionId)
  if (!targetGroup) {
    activeSidebarGroup.value = normalizedSectionId === 'dashboard' ? 'dashboard' : ''
  } else if (targetGroup.id === 'dashboard') {
    activeSidebarGroup.value = 'dashboard'
  } else {
    activeSidebarGroup.value = targetGroup.id
    if (!expandedSidebarGroups.value.includes(targetGroup.id)) {
      expandedSidebarGroups.value = [...expandedSidebarGroups.value, targetGroup.id]
    }
  }

  closeFloatingMenus()
}

const openSidebarGroup = (group = {}) => {
  const groupId = text(group.id)
  if (!groupId) return

  if (groupId === 'dashboard') {
    syncSection('dashboard')
    return
  }

  activeSidebarGroup.value = groupId
  expandedSidebarGroups.value = expandedSidebarGroups.value.includes(groupId)
    ? expandedSidebarGroups.value.filter((entry) => entry !== groupId)
    : [...expandedSidebarGroups.value, groupId]
}

const handleSidebarSectionClick = (item = {}) => {
  syncSection(text(item.id) || 'dashboard')
}

const openBusinessSubscriptionSection = () => {
  syncSection('subscriptions')
}

const openBusinessProfileSection = () => {
  syncSection('profile')
}

const toggleBusinessProfileMenu = () => {
  isProfileMenuOpen.value = !isProfileMenuOpen.value
  if (isProfileMenuOpen.value) {
    isNotificationMenuOpen.value = false
  }
}

const toggleNotificationMenu = () => {
  isNotificationMenuOpen.value = !isNotificationMenuOpen.value
  if (isNotificationMenuOpen.value) {
    isProfileMenuOpen.value = false
  }
}

const openBusinessNotification = (notificationId = '') => {
  const normalizedId = text(notificationId)
  const targetNotification = businessNavbarNotifications.value.find((item) => item.id === normalizedId) || null
  if (!targetNotification) return

  businessNavbarNotifications.value = businessNavbarNotifications.value.map((item) =>
    item.id === normalizedId
      ? { ...item, read: true }
      : item,
  )

  isNotificationMenuOpen.value = false
  syncSection(text(targetNotification.targetSection) || 'dashboard')
}

const performLogout = async () => {
  if (isLogoutSubmitting.value) return

  isLogoutConfirmOpen.value = false
  isLogoutSubmitting.value = true
  clearAuthSession()

  if (typeof window !== 'undefined') {
    sessionStorage.setItem('showLoggedOutToast', '1')
  }

  try {
    await router.replace('/login')
  } finally {
    isLogoutSubmitting.value = false
  }
}

const openLogoutConfirm = () => {
  isProfileMenuOpen.value = false
  isNotificationMenuOpen.value = false
  isLogoutConfirmOpen.value = true
}

const closeLogoutConfirm = () => {
  if (isLogoutSubmitting.value) return
  isLogoutConfirmOpen.value = false
}

const handleBusinessNavbarSetting = async (label = '') => {
  const normalizedLabel = text(label).toLowerCase()
  if (normalizedLabel === 'profile') {
    openBusinessProfileSection()
    return
  }

  if (normalizedLabel === 'subscriptions') {
    openBusinessSubscriptionSection()
    return
  }

  if (normalizedLabel === 'log out' || normalizedLabel === 'logout') {
    await openLogoutConfirm()
  }
}

const isPremiumGuideTarget = () => false

const handleDocumentClick = () => {
  if (!isProfileMenuOpen.value && !isNotificationMenuOpen.value) return
  closeFloatingMenus()
}

onMounted(() => {
  syncDashboardSubscriptions()
  syncProfileFormFromStoredUser()
  if (typeof document !== 'undefined') {
    document.addEventListener('click', handleDocumentClick)
  }
})

watch(
  () => [
    businessWorkspaceOwnerId.value,
    businessWorkspaceOwnerEmailForSync.value,
    storedUserId.value,
  ],
  () => {
    syncDashboardSubscriptions()
  },
)

watch(
  storedUser,
  () => {
    syncProfileFormFromStoredUser()
  },
  { deep: true },
)

onBeforeUnmount(() => {
  stopDashboardSyncSubscriptions()
  if (typeof document !== 'undefined') {
    document.removeEventListener('click', handleDocumentClick)
  }
})
</script>

<template>
  <section class="business-shell">
    <div class="business-shell__sidebar">
      <component
        :is="activeSidebarComponent"
        :show-business-sidebar="true"
        :pwd-logo="pwdLogo"
        :business-name="businessSidebarBrandName"
        :business-brand-subtitle="businessSidebarBrandSubtitle"
        :profile-name="loggedInBusinessUserName"
        :profile-avatar="loggedInBusinessUserAvatar"
        :profile-initials="loggedInBusinessUserInitials"
        :profile-email="loggedInBusinessUserEmail"
        :profile-role-label="loggedInBusinessUserRoleLabel"
        :secondary-section-label="businessSidebarSecondarySectionLabel"
        :show-subscription-only-sidebar="false"
        :show-business-subscription-shortcut="showBusinessSubscriptionShortcut"
        :show-business-profile-shortcut="showBusinessProfileShortcut"
        :sidebar-render-groups="sidebarRenderGroups"
        :has-dashboard-sidebar-link="hasDashboardSidebarLink"
        :active-section="activeSection"
        :open-sidebar-group="openSidebarGroup"
        :handle-sidebar-section-click="handleSidebarSectionClick"
        :get-sidebar-item-icon="getSidebarItemIcon"
        :is-premium-guide-target="isPremiumGuideTarget"
        :is-logout-submitting="isLogoutSubmitting"
        :open-logout-confirm="openLogoutConfirm"
        :open-business-subscription-section="openBusinessSubscriptionSection"
        :open-business-profile-section="openBusinessProfileSection"
      />
    </div>

    <div class="business-shell__workspace">
      <div class="business-shell__navbar">
        <component
          :is="activeNavbarComponent"
          :current-section="currentSection"
          :breadcrumb-parent="businessNavbarBreadcrumbParent"
          :business-name="loggedInBusinessUserName"
          :business-email="loggedInBusinessUserEmail"
          :business-initials="loggedInBusinessUserInitials"
          :profile-avatar="loggedInBusinessUserAvatar"
          :profile-role-label="loggedInBusinessUserRoleLabel"
          :workspace-owner-name="businessWorkspaceOwnerName"
          :is-profile-menu-open="isProfileMenuOpen"
          :business-navbar-settings-items="businessNavbarSettingsItems"
          :business-navbar-notifications="businessNavbarNotifications"
          :unread-business-notification-count="unreadBusinessNotificationCount"
          :is-notification-menu-open="isNotificationMenuOpen"
          :toggle-business-profile-menu="toggleBusinessProfileMenu"
          :toggle-notification-menu="toggleNotificationMenu"
          :open-business-notification="openBusinessNotification"
          :handle-business-navbar-setting="handleBusinessNavbarSetting"
          :open-logout-confirm="openLogoutConfirm"
        />
      </div>

      <main class="business-shell__content">
        <transition name="business-shell-stage" mode="out-in">
          <section :key="activeSection" class="business-shell__stage">
            <BusinessDashboardStage
              v-if="activeSection === 'dashboard'"
              :business-category="businessSidebarBrandSubtitle"
              :business-name="businessSidebarBrandName"
              :current-section="currentSection"
              :has-unlocked-business-workspace="true"
              :dashboard-sync-overview="dashboardSyncOverview"
              :dashboard-applicant-feed="dashboardApplicantFeed"
              :summary-cards="dashboardSummaryCards"
              :business-trend-chart="businessTrendChart"
              :dashboard-progress-cards="[]"
              :dashboard-bar-series="dashboardBarSeries"
              :dashboard-donut-legend="dashboardDonutLegend"
              :dashboard-donut-style="dashboardDonutStyle"
              :dashboard-donut-total="dashboardDonutTotal"
            />

            <BusinessJobPostingStage
              v-else-if="activeSection === 'job-posting'"
              :workspace-owner-id="workspaceOwnerId"
              :workspace-owner-email="workspaceOwnerEmail"
              :business-name="businessSidebarBrandName"
              :business-category="businessProfileCategory"
              :business-avatar="loggedInBusinessUserAvatar"
              :actor-id="storedUserId"
              :actor-name="loggedInBusinessUserName"
              :is-employee-workspace-mode="isBusinessEmployeeWorkspaceMode"
            />

            <BusinessApplicantManagementStage
              v-else-if="activeSection === 'applicant-management'"
              :business-job-applications="businessJobApplications"
            />

            <BusinessIssueOfferStage
              v-else-if="activeSection === 'issue-offer'"
              :business-job-applications="businessJobApplications"
              :is-employee-workspace-mode="isBusinessEmployeeWorkspaceMode"
            />

            <BusinessContractSigningStage
              v-else-if="activeSection === 'contract-signing'"
              :business-job-applications="businessJobApplications"
              :business-name="businessProfileName"
              :workspace-owner-id="businessWorkspaceOwnerId"
              :workspace-owner-email="businessWorkspaceOwnerEmailForSync"
              :is-employee-workspace-mode="isBusinessEmployeeWorkspaceMode"
            />

            <BusinessAssessmentStage
              v-else-if="activeSection === 'assessment-management' || activeSection === 'assessment-assignment' || activeSection === 'applicant-score'"
              :active-section="activeSection"
              :business-job-applications="businessJobApplications"
              :workspace-owner-id="businessWorkspaceOwnerId"
              :business-name="businessProfileName"
              :is-employee-workspace-mode="isBusinessEmployeeWorkspaceMode"
            />

            <BusinessInterviewStage
              v-else-if="activeSection === 'interview-scheduling' || activeSection === 'interview-status'"
              :active-section="activeSection"
              :business-job-applications="businessJobApplications"
              :workspace-owner-id="businessWorkspaceOwnerId"
              :business-name="businessProfileName"
            />

            <BusinessTrainingStage
              v-else-if="activeSection === 'training-templates'"
              :employee-directory="employeeDirectory"
            />

            <BusinessTeamMembersStage
              v-else-if="activeSection === 'user-overview' || activeSection === 'create-user' || activeSection === 'add-employee' || activeSection === 'permissions'"
              :active-section="activeSection"
              :workspace-user-directory="workspaceUserDirectory"
              :employee-directory="employeeDirectory"
            />

            <section v-else-if="activeSection === 'profile'" class="business-profile">
              <div class="business-profile__header">
                <div>
                  <h2>Business Profile</h2>
                  <p>Manage the details shown for your business account and workspace identity.</p>
                </div>
                <div class="business-profile__badge">
                  <i class="bi bi-buildings" aria-hidden="true" />
                  <span>{{ isBusinessEmployeeWorkspaceMode ? 'Workspace Account' : 'Business Account' }}</span>
                </div>
              </div>

              <div
                v-if="profileStatusMessage"
                class="business-shell__panel"
                :class="profileStatusTone === 'error' ? 'business-shell__panel--error' : 'business-shell__panel--success'"
              >
                <strong>{{ profileStatusTone === 'error' ? 'Profile Update Failed' : 'Profile Ready' }}</strong>
                <p>{{ profileStatusMessage }}</p>
              </div>

              <div class="business-profile__grid">
                <article class="business-profile__card">
                  <h3>Company Information</h3>
                  <div class="business-profile__avatar-section">
                    <div class="business-profile__avatar-shell" :class="{ 'is-loading': isProfileAvatarLoading }">
                      <img
                        v-if="profileForm.avatar"
                        :src="profileForm.avatar"
                        alt="Business profile avatar"
                        class="business-profile__avatar-image"
                        :class="{ 'is-visible': profileForm.avatar }"
                      />
                      <div v-if="isProfileAvatarLoading" class="business-profile__avatar-loader" aria-hidden="true" />
                      <span v-else-if="!profileForm.avatar">{{ businessProfileInitials }}</span>
                    </div>
                    <div class="business-profile__avatar-copy">
                      <strong>Profile Photo</strong>
                      <span>Upload your business image or logo to update the account avatar.</span>
                    </div>
                    <div class="business-profile__avatar-actions">
                      <input
                        ref="profileAvatarInputRef"
                        type="file"
                        accept="image/png,image/jpeg,image/jpg,image/webp"
                        class="business-profile__avatar-input"
                        @change="handleProfileAvatarChange"
                      />
                      <button type="button" class="business-profile__secondary" :disabled="isProfileAvatarLoading" @click="openProfileAvatarPicker">
                        {{ isProfileAvatarLoading ? 'Loading image...' : profileForm.avatar ? 'Change Photo' : 'Upload Photo' }}
                      </button>
                      <button
                        v-if="profileForm.avatar"
                        type="button"
                        class="business-profile__secondary business-profile__secondary--danger"
                        :disabled="isProfileAvatarLoading"
                        @click="removeProfileAvatar"
                      >
                        Remove Photo
                      </button>
                    </div>
                  </div>

                  <div class="business-profile__form">
                    <label class="business-profile__field">
                      <span>Company Name</span>
                      <input v-model="profileForm.companyName" type="text" placeholder="Enter company name" />
                    </label>
                    <label class="business-profile__field">
                      <span>Business Email</span>
                      <input v-model="profileForm.email" type="email" placeholder="Enter business email" />
                    </label>
                    <label class="business-profile__field">
                      <span>Category</span>
                      <input v-model="profileForm.category" type="text" placeholder="Enter business category" />
                    </label>
                    <label class="business-profile__field">
                      <span>Location</span>
                      <input v-model="profileForm.location" type="text" placeholder="Enter business location" />
                    </label>
                    <label class="business-profile__field business-profile__field--full">
                      <span>Contact Person</span>
                      <input v-model="profileForm.contactPerson" type="text" placeholder="Enter contact person name" />
                    </label>
                  </div>

                  <div class="business-profile__actions">
                    <button type="button" class="business-profile__secondary" @click="syncProfileFormFromStoredUser">Reset</button>
                    <button type="button" class="business-profile__primary" @click="saveBusinessProfile">Save Changes</button>
                  </div>
                </article>

                <aside class="business-profile__card business-profile__card--preview">
                  <h3>Profile Preview</h3>
                  <div class="business-profile__preview">
                    <div class="business-profile__preview-avatar" :class="{ 'is-loading': isProfileAvatarLoading }">
                      <img
                        v-if="profileForm.avatar"
                        :src="profileForm.avatar"
                        alt="Business profile preview avatar"
                        class="business-profile__preview-avatar-image"
                        :class="{ 'is-visible': profileForm.avatar }"
                      />
                      <template v-else>{{ businessProfileInitials }}</template>
                    </div>
                    <strong>{{ businessProfileName }}</strong>
                    <span>{{ businessProfileEmail || 'No business email set' }}</span>
                  </div>

                  <div class="business-profile__meta">
                    <div class="business-profile__meta-row">
                      <span>Category</span>
                      <strong>{{ businessProfileCategory }}</strong>
                    </div>
                    <div class="business-profile__meta-row">
                      <span>Location</span>
                      <strong>{{ businessProfileLocation }}</strong>
                    </div>
                    <div class="business-profile__meta-row">
                      <span>Contact Person</span>
                      <strong>{{ businessProfileContactPerson }}</strong>
                    </div>
                  </div>
                </aside>
              </div>
            </section>

            <BusinessSubscriptionStage
              v-else-if="activeSection === 'subscriptions'"
              :business-name="businessSidebarBrandName"
              :business-email="loggedInBusinessUserEmail"
            />

            <template v-else>
              <section class="business-shell__panel">
              <strong>Placeholder mode is active</strong>
              <p>
                Business login, sidebar navigation, and account layout are working again. The heavy
                backend-driven screens stay disabled so you can debug the rest of the app safely.
              </p>
            </section>

            <section class="business-shell__grid" aria-label="Business placeholder sections">
              <button
                v-for="(item, index) in visibleSectionCards"
                :key="item.id"
                type="button"
                class="business-shell__card business-shell__card--motion"
                :class="{ 'is-active': activeSection === item.id }"
                :style="{ '--business-card-delay': `${210 + (index * 55)}ms` }"
                @click="syncSection(item.id)"
              >
                <span class="business-shell__card-icon">
                  <i :class="getSidebarItemIcon(item.id)" aria-hidden="true" />
                </span>
                <strong>{{ item.label }}</strong>
                <p>{{ sectionMeta[item.id]?.description || 'Business section placeholder.' }}</p>
              </button>
            </section>
            </template>
          </section>
        </transition>
      </main>
    </div>

    <transition name="business-logout-confirm">
      <div
        v-if="isLogoutConfirmOpen"
        class="business-logout-confirm"
        role="dialog"
        aria-modal="true"
        aria-labelledby="business-logout-confirm-title"
      >
        <div class="business-logout-confirm__backdrop" @click="closeLogoutConfirm" />
        <div class="business-logout-confirm__card">
          <div class="business-logout-confirm__icon" aria-hidden="true">
            <i class="bi bi-box-arrow-right" />
          </div>
          <div class="business-logout-confirm__copy">
            <p class="business-logout-confirm__eyebrow">Business Workspace</p>
            <h2 id="business-logout-confirm-title">Log out from this workspace?</h2>
            <p>Your current Business session will close and you will return to the login screen.</p>
          </div>
          <div class="business-logout-confirm__actions">
            <button
              type="button"
              class="business-logout-confirm__button business-logout-confirm__button--ghost"
              :disabled="isLogoutSubmitting"
              @click="closeLogoutConfirm"
            >
              Cancel
            </button>
            <button
              type="button"
              class="business-logout-confirm__button business-logout-confirm__button--primary"
              :disabled="isLogoutSubmitting"
              @click="performLogout"
            >
              {{ isLogoutSubmitting ? 'Logging Out...' : 'Log Out' }}
            </button>
          </div>
        </div>
      </div>
    </transition>
  </section>
</template>

<style scoped>
.business-shell {
  min-height: 100vh;
  display: grid;
  grid-template-columns: minmax(18.5rem, 20.5rem) minmax(0, 1fr);
  font-family: "Inter", "Segoe UI", sans-serif;
  background:
    radial-gradient(circle at top right, rgba(151, 191, 166, 0.18), transparent 28rem),
    linear-gradient(180deg, #eef5ef 0%, #f7faf8 100%);
}

.business-shell__sidebar {
  min-width: 0;
}

.business-shell__workspace {
  min-width: 0;
  display: grid;
  grid-template-rows: auto 1fr;
  height: 100vh;
  max-height: 100vh;
  overflow: hidden;
  padding: 1.35rem 1.35rem 1.5rem;
  gap: 1rem;
}

.business-shell__navbar {
  position: sticky;
  top: 0;
  z-index: 160;
}

.business-shell__content {
  display: grid;
  align-content: start;
  gap: 1rem;
  overflow-y: auto;
  padding-right: 0.35rem;
  scrollbar-gutter: stable;
  min-height: 0;
  height: 100%;
}

.business-shell__stage {
  display: grid;
  align-content: start;
  gap: 1rem;
  min-width: 0;
  min-height: 0;
  will-change: opacity, transform;
}

.business-shell-stage-enter-active,
.business-shell-stage-leave-active {
  transition:
    opacity 0.34s cubic-bezier(0.22, 1, 0.36, 1),
    transform 0.34s cubic-bezier(0.22, 1, 0.36, 1);
}

.business-shell-stage-enter-from,
.business-shell-stage-leave-to {
  opacity: 0;
  transform: translate3d(0, 8px, 0);
}

@keyframes business-shell-card-enter {
  0% {
    opacity: 0;
    transform: translate3d(0, 8px, 0);
  }

  100% {
    opacity: 1;
    transform: translate3d(0, 0, 0);
  }
}

.business-shell__panel,
.business-shell__card--motion {
  opacity: 0;
  transform: translate3d(0, 8px, 0);
  animation: business-shell-card-enter 480ms cubic-bezier(0.22, 1, 0.36, 1) forwards;
  animation-delay: var(--business-card-delay, 0ms);
  will-change: transform, opacity;
}

.business-shell__panel p,
.business-shell__card p {
  margin: 0;
}
.business-shell__panel p,
.business-shell__card p {
  line-height: 1.6;
  color: #496052;
}

.business-shell__panel {
  display: grid;
  gap: 0.55rem;
  padding: 1.2rem 1.3rem;
  border: 1px solid rgba(204, 217, 209, 0.95);
  border-radius: 22px;
  background: rgba(241, 247, 243, 0.92);
  contain: layout paint;
  content-visibility: auto;
  contain-intrinsic-size: 1px 8rem;
  --business-card-delay: 140ms;
}

.business-shell__panel--success {
  border-color: rgba(170, 210, 186, 0.95);
  background: rgba(242, 251, 246, 0.96);
}

.business-shell__panel--error {
  border-color: rgba(232, 192, 192, 0.95);
  background: rgba(253, 244, 244, 0.96);
}

.business-shell__panel strong,
.business-shell__card strong {
  color: #183123;
}

.business-shell__grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(14rem, 1fr));
  gap: 1rem;
  contain: layout paint;
  content-visibility: auto;
  contain-intrinsic-size: 1px 26rem;
}

.business-shell__card {
  display: grid;
  gap: 0.85rem;
  padding: 1.2rem;
  border: 1px solid rgba(205, 218, 210, 0.95);
  border-radius: 24px;
  background: rgba(255, 255, 255, 0.96);
  text-align: left;
  cursor: pointer;
  transition: transform 0.18s ease, box-shadow 0.18s ease, border-color 0.18s ease;
}

.business-logout-confirm {
  position: fixed;
  inset: 0;
  z-index: 2500;
  display: grid;
  place-items: center;
  padding: 1.5rem;
}

.business-logout-confirm__backdrop {
  position: absolute;
  inset: 0;
  background: rgba(15, 23, 42, 0.34);
  backdrop-filter: blur(3px);
}

.business-logout-confirm__card {
  position: relative;
  z-index: 1;
  width: min(100%, 28rem);
  display: grid;
  gap: 1rem;
  padding: 1.35rem;
  border: 1px solid rgba(221, 227, 235, 0.96);
  border-radius: 1.35rem;
  background: rgba(255, 255, 255, 0.98);
  box-shadow: 0 24px 60px rgba(15, 23, 42, 0.18);
}

.business-logout-confirm__icon {
  width: 3rem;
  height: 3rem;
  display: grid;
  place-items: center;
  border-radius: 1rem;
  background: linear-gradient(135deg, #eef3fb 0%, #f8fbff 100%);
  color: #35507d;
  font-size: 1.2rem;
}

.business-logout-confirm__eyebrow {
  margin: 0 0 0.32rem;
  color: #66748b;
  font-size: 0.74rem;
  font-weight: 800;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.business-logout-confirm__copy h2,
.business-logout-confirm__copy p {
  margin: 0;
}

.business-logout-confirm__copy h2 {
  color: #172033;
  font-size: 1.18rem;
  font-weight: 800;
}

.business-logout-confirm__copy p:last-child {
  margin-top: 0.45rem;
  color: #607088;
  line-height: 1.6;
}

.business-logout-confirm__actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.business-logout-confirm__button {
  min-width: 8.4rem;
  min-height: 2.9rem;
  border: 1px solid rgba(218, 225, 236, 0.96);
  border-radius: 999px;
  padding: 0 1.15rem;
  font: inherit;
  font-size: 0.92rem;
  font-weight: 700;
  cursor: pointer;
  transition: transform 0.18s ease, box-shadow 0.18s ease, background-color 0.18s ease, border-color 0.18s ease;
}

.business-logout-confirm__button--ghost {
  background: #f7f9fc;
  color: #54647b;
}

.business-logout-confirm__button--primary {
  background: linear-gradient(135deg, #2d5b9b 0%, #466db2 100%);
  border-color: rgba(70, 109, 178, 0.9);
  color: #ffffff;
  box-shadow: 0 14px 28px rgba(70, 109, 178, 0.22);
}

.business-logout-confirm__button:hover:not(:disabled) {
  transform: translateY(-1px);
}

.business-logout-confirm__button:disabled {
  cursor: wait;
  opacity: 0.72;
}

.business-logout-confirm-enter-active,
.business-logout-confirm-leave-active {
  transition: opacity 0.2s ease;
}

.business-logout-confirm-enter-active .business-logout-confirm__card,
.business-logout-confirm-leave-active .business-logout-confirm__card {
  transition: transform 0.24s ease, opacity 0.24s ease;
}

.business-logout-confirm-enter-from,
.business-logout-confirm-leave-to {
  opacity: 0;
}

.business-logout-confirm-enter-from .business-logout-confirm__card,
.business-logout-confirm-leave-to .business-logout-confirm__card {
  opacity: 0;
  transform: translateY(10px) scale(0.98);
}

.business-shell__card:hover,
.business-shell__card.is-active {
  transform: translateY(-2px);
  border-color: #8eb49a;
  box-shadow: 0 18px 38px rgba(33, 67, 48, 0.08);
}

.business-shell__card-icon {
  width: 2.7rem;
  height: 2.7rem;
  display: grid;
  place-items: center;
  border-radius: 16px;
  background: #edf5ef;
  color: #25553a;
  font-size: 1.15rem;
}

@media (prefers-reduced-motion: reduce) {
  .business-shell-stage-enter-active,
  .business-shell-stage-leave-active {
    transition: none !important;
  }

  .business-shell__panel,
  .business-shell__card--motion {
    opacity: 1 !important;
    transform: none !important;
    animation: none !important;
  }
}

@media (max-width: 1080px) {
  .business-shell {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 720px) {
  .business-shell__workspace {
    padding: 1rem;
  }

}
</style>
