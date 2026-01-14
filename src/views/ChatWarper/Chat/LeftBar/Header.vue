<template>
  <div class="flex-shrink-0 px-2 gap-1 flex justify-start items-center">
    <div
      v-if="current_group_name"
      v-tooltip.bottom="getPageNames(current_group_pages)"
      class="font-semibold text-2xl truncate cursor-pointer"
    >
      {{ current_group_name }}
    </div>
    <div
      v-else-if="selected_platform_group"
      v-tooltip.bottom="getPageNames(selected_platform_group.pages)"
      class="font-semibold text-2xl truncate cursor-pointer"
    >
      <div class="flex items-center gap-1">
        {{ $t(`v1.common.${selected_platform_group.type?.toLowerCase()}`) }}
      </div>
    </div>
    <div
      v-else-if="is_show_all_groups"
      v-tooltip.bottom="getPageNames(all_selected_pages)"
      class="font-semibold text-2xl truncate cursor-pointer"
    >
      {{ $t('Tất cả nhóm') }}
    </div>
    <div
      v-else-if="adhoc_page_group"
      v-tooltip.bottom="getPageNames(adhoc_page_group)"
      class="font-semibold text-2xl truncate cursor-pointer"
    >
      {{ $t('Gộp') }} {{ adhoc_page_group.length }} {{ $t('trang') }}
    </div>
    <div
      v-else
      v-tooltip.bottom="`v${version}`"
      class="font-semibold text-2xl truncate"
    >
      <template v-if="orgStore.selected_org_info?.org_info?.org_name">
        {{ orgStore.selected_org_info?.org_info?.org_name }}
      </template>
      <template v-else>
        {{ commonStore.partner?.name }}
      </template>
    </div>
    <!-- <Badge
      v-if="count_all_unread"
      :value="count_all_unread"
    /> -->
  </div>
  <div class="flex-shrink-0 flex items-center justify-between">
    <template v-if="!is_search">
      <div class="text-sm gap-3 flex items-center h-8">
        <button
          @click="$main.activeTab('CHAT')"
          :class="{
            'font-semibold border-b-2 border-black':
              !conversationStore.option_filter_page_data.conversation_type ||
              conversationStore.option_filter_page_data.conversation_type ===
                'CHAT',
          }"
          class="h-full flex gap-1 items-center"
        >
          <p>{{ $t('Chat') }}</p>
          <p
            v-if="conversationStore.count_conversation?.chat"
            class="rounded-full bg-slate-200 text-slate-700 font-medium text-xxs px-1 leading-[14px] max-w-20 truncate"
          >
            {{ currency(conversationStore.count_conversation.chat) }}
          </p>
        </button>
        <button
          @click="$main.activeTab('POST')"
          :class="{
            'font-semibold border-b-2 border-black':
              conversationStore.option_filter_page_data.conversation_type ===
              'POST',
          }"
          class="h-full flex gap-1 items-center"
        >
          <p>{{ $t('Bài viết') }}</p>
          <p
            v-if="conversationStore.count_conversation?.post"
            class="rounded-full bg-slate-200 text-slate-700 font-medium text-xxs px-1 leading-[14px] max-w-20 truncate"
          >
            {{ currency(conversationStore.count_conversation.post) }}
          </p>
        </button>
      </div>
      <div class="flex gap-3 text-slate-500">
        <button
          v-show="
            conversationStore.select_conversation?.platform_type?.includes(
              'ZALO'
            )
          "
          class="p-2 bg-slate-100 rounded-full"
          @click="
            () => {
              modal_zalo_create_group_ref?.toggleModal()
              message_data = undefined
            }
          "
          v-tooltip="$t('v1.common.create_new_group')"
        >
          <UserGroupIcon class="size-4 flex-shrink-0" />
        </button>
        <button
          class="p-2 bg-slate-100 rounded-full"
          @click="
            () => {
              modal_zalo_personal_ref?.toggleModal()
              message_data = undefined
            }
          "
          v-tooltip="$t('v1.common.add_customer')"
        >
          <UserPlusIcon class="size-4 flex-shrink-0" />
        </button>
        <button
          @click="$main.toggleSearch()"
          class="w-8 h-8 bg-slate-100 rounded-full flex justify-center items-center"
          v-tooltip="$t('v1.common.search')"
        >
          <SearchIcon class="w-4 h-4" />
        </button>
      </div>
    </template>
    <div
      v-else
      class="relative w-full"
    >
      <SearchIcon
        class="absolute top-1/2 left-3 -translate-y-1/2 w-4 h-4 text-slate-500"
      />
      <input
        v-model="search_conversation"
        @blur="$main.toggleSearch()"
        ref="ref_search_conversation"
        class="w-full bg-slate-100 placeholder-slate-500 py-1.5 pl-9 pr-8 text-sm rounded-full"
        type="text"
        :placeholder="$t('v1.common.search')"
      />
      <XCircleIcon
        @click="search_conversation = undefined"
        v-if="search_conversation"
        class="absolute top-1/2 right-2 -translate-y-1/2 size-5 text-red-500 cursor-pointer"
      />
    </div>
  </div>
  <div
    v-if="isFilterActive()"
    class="bg-slate-100 rounded-lg py-1.5 px-2 text-xs flex gap-2 items-center"
    :class="{
      hidden: conversationStore.selected_quick_filter !== 'ALL',
    }"
  >
    <div class="flex gap-2 w-full min-w-0">
      <FunnelIcon class="w-3.5 h-3.5 flex-shrink-0" />
      <p class="truncate">{{ filter }}</p>
    </div>
    <button @click="$filter_service.clearAllFilter()">
      <XMarkIcon class="w-3.5 h-3.5 flex-shrink-0" />
    </button>
  </div>
  <!-- <QuickFilter /> -->
</template>
<script setup lang="ts">
import { isFilterActive } from '@/service/function'
import { currency } from '@/service/helper/format'
import {
  useCommonStore,
  useConversationStore,
  useMessageStore,
  useOrgStore,
  usePageStore,
} from '@/stores'
import { FilterService } from '@/utils/helper/Filter'
import { BillingAppGroup } from '@/utils/api/Billing'
import { format } from 'date-fns'
import { debounce, map, values } from 'lodash'
import { container } from 'tsyringe'
import { computed, nextTick, onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRouter } from 'vue-router'

import QuickFilter from '@/views/ChatWarper/Chat/LeftBar/Header/QuickFilter.vue'

import SearchIcon from '@/components/Icons/Search.vue'
import {
  FunnelIcon,
  UserGroupIcon,
  UserPlusIcon,
  XMarkIcon,
} from '@heroicons/vue/24/outline'
import { XCircleIcon } from '@heroicons/vue/24/solid'

import type { ILabel } from '@/service/interface/app/label'
import type { PageData } from '@/service/interface/app/page'
import type { StaffInfo } from '@/service/interface/app/staff'
import { storeToRefs } from 'pinia'

/**tab đang kích hoạt */
type IActiveTab = 'CHAT' | 'POST'

/** lấy tên các trang */
function getPageNames(pages?: PageData[]) {
  if (!pages?.length) return ''

  /** số lượng trang tối đa hiển thị */
  const MAX_DISPLAY = 15

  /** danh sách trang hiển thị */
  const display_pages = pages
    .slice(0, MAX_DISPLAY)
    .map(p => p.page?.name)
    .join(', ')

  if (pages.length > MAX_DISPLAY) {
    return `${display_pages}, ... ${$t('và')} ${
      pages.length - MAX_DISPLAY
    } ${$t('trang khác')}`
  }

  return display_pages
}

const conversationStore = useConversationStore()
const commonStore = useCommonStore()
const pageStore = usePageStore()
const orgStore = useOrgStore()
// i18n
const { t: $t } = useI18n()
// props
defineProps<{
  /** có nên hiển thị skeleton loading ko */
  is_loading?: boolean
}>()

const { modal_zalo_personal_ref, message_data, modal_zalo_create_group_ref } =
  storeToRefs(useMessageStore())

/** router */
const $router = useRouter()

const $filter_service = container.resolve(FilterService)

/**phiên bản trong package.json */
const version = npm_package_version
/**giá trị của ô tìm kiếm hội thoại */
const search_conversation = ref<string>()
/**trạng thái tìm kiếm */
const is_search = ref<boolean>(
  !!conversationStore.option_filter_page_data.search
)
/**tham chiếu đến ô tìm kiếm */
const ref_search_conversation = ref<HTMLInputElement>()

/**delay tìm kiếm hội thoại */
const onSearchConversation = debounce((value?: string) => {
  // lưu giá trị search vào biến
  conversationStore.option_filter_page_data.search = value
}, 300)

/** dữ liệu lọc thể hiện ra dạng chuỗi */
const filter = computed(() => {
  /** dữ liệu lọc chung */
  const FILTER_GENERAL: string[] = []
  /** lọc gắn nhãn */
  const FILTER_TAG: string[] = []
  /** lọc trừ nhãn */
  const FILTER_NOT_TAG: string[] = []
  /** lọc thời gian */
  const FITLER_TIME: string[] = []
  /** lọc nhân sự */
  const FILTER_STAFF: string[] = []

  /** nếu là lọc tương tác từ tin nhắn */
  if (conversationStore.option_filter_page_data.display_style === 'INBOX') {
    FILTER_GENERAL.push(
      $t('v1.view.main.dashboard.chat.filter.interact.message')
    )
  }
  /** nếu là lọc tương tác từ bình luận */
  if (conversationStore.option_filter_page_data.display_style === 'COMMENT') {
    FILTER_GENERAL.push(
      $t('v1.view.main.dashboard.chat.filter.interact.comment')
    )
  }
  /** nếu là lọc tương tác từ bạn bè */
  if (conversationStore.option_filter_page_data.display_style === 'FRIEND') {
    FILTER_GENERAL.push(
      $t('v1.view.main.dashboard.chat.filter.interact.friend')
    )
  }
  /** nếu là lọc tương tác từ nhóm */
  if (conversationStore.option_filter_page_data.display_style === 'GROUP') {
    FILTER_GENERAL.push($t('v1.view.main.dashboard.chat.filter.interact.group'))
  }

  /** nếu là lọc chưa đọc */
  if (conversationStore.option_filter_page_data.unread_message === 'true') {
    FILTER_GENERAL.push($t('v1.view.main.dashboard.chat.filter.message.unread'))
  }
  /** nếu là lọc chưa phản hồi */
  if (
    conversationStore.option_filter_page_data.not_response_client === 'true'
  ) {
    FILTER_GENERAL.push(
      $t('v1.view.main.dashboard.chat.filter.message.not_reply')
    )
  }
  /** nếu là lọc tin nhắn chứa gắn nhãn */
  if (conversationStore.option_filter_page_data.not_exist_label === 'true') {
    FILTER_GENERAL.push(
      $t('v1.view.main.dashboard.chat.filter.message.not_tag')
    )
  }
  /** nếu là lọc tin nhắn spam */
  if (conversationStore.option_filter_page_data.is_spam_fb === 'YES') {
    FILTER_GENERAL.push($t('v1.view.main.dashboard.chat.filter.message.spam'))
  }
  /** nếu là lọc có số điện thoại */
  if (conversationStore.option_filter_page_data.have_phone === 'YES') {
    FILTER_GENERAL.push(
      $t('v1.view.main.dashboard.chat.filter.phone.include_phone')
    )
  }
  /** nếu là lọc không có số điện thoại */
  if (conversationStore.option_filter_page_data.have_phone === 'NO') {
    FILTER_GENERAL.push(
      $t('v1.view.main.dashboard.chat.filter.phone.exclude_phone')
    )
  }
  /** nếu là lọc ngày */
  if (conversationStore.option_filter_page_data.time_range) {
    /** thời điểm bắt đầu lọc */
    const START = conversationStore.option_filter_page_data?.time_range?.gte
    /** thời điểm kết thúc lọc */
    const END = conversationStore.option_filter_page_data?.time_range?.lte

    // nếu có thì mới thêm vào
    if (START && END) {
      FITLER_TIME.push(
        `${format(START, 'HH:mm, dd/MM/yyyy')} - ${format(
          END,
          'HH:mm, dd/MM/yyyy'
        )}`
      )
    }
  }
  /** nếu là lọc nhãn */
  if (conversationStore.option_filter_page_data.label_id?.length) {
    /** danh sách các tiêu đề nhãn đã chọn */
    const TITLE_TAGS = conversationStore.option_filter_page_data.label_id?.map(
      id => tags.value?.[id]?.title || ''
    )

    FILTER_TAG.push(...TITLE_TAGS)
  }
  /** nếu là lọc trừ nhãn */
  if (conversationStore.option_filter_page_data.not_label_id?.length) {
    /** danh sách các tiêu đề nhãn trừ nhãn */
    const TITLE_NOT_TAGS =
      conversationStore.option_filter_page_data.not_label_id?.map(
        id => tags.value?.[id]?.title || ''
      )

    FILTER_NOT_TAG.push(...TITLE_NOT_TAGS)
  }
  /** nếu là lọc nhân sự */
  if (conversationStore.option_filter_page_data.staff_id?.length) {
    /** danh sách tên các nhân sự được lọc */
    const STAFF_NAMES = conversationStore.option_filter_page_data.staff_id?.map(
      id => staffs.value?.[id]?.name || ''
    )

    FILTER_STAFF.push(...STAFF_NAMES)
  }

  /** nội dung của các bộ lọc */
  const RESULT: string[] = []
  /** thêm nội dung lọc chung */
  addContent(
    RESULT,
    FILTER_GENERAL,
    $t('v1.view.main.dashboard.chat.filter.post.filter')
  )
  /** thêm nội dung lọc nhãn */
  addContent(RESULT, FILTER_TAG, $t('Nhãn'))
  /** thêm nội dung lọc trừ nhãn */
  addContent(RESULT, FILTER_NOT_TAG, $t('Trừ nhãn'))
  /** thêm nội dung lọc thời gian */
  addContent(RESULT, FITLER_TIME, $t('Thời gian'))
  /** thêm nội dung lọc nhân sự */
  addContent(RESULT, FILTER_STAFF, $t('Nhân viên'))
  /** thêm lọc bài viết */
  if (conversationStore.option_filter_page_data.post_id) {
    RESULT.push($t('Lọc bài viết'))
  }
  return RESULT.join(', ')
})

/** nhóm trang theo nền tảng được chọn */
const selected_platform_group = computed(() => {
  /** danh sách trang đã chọn */
  const list_page = values(pageStore.selected_page_list_info)

  /** nếu chỉ chọn 1 trang thì hiển thị tên tổ chức như cũ */
  if (list_page.length <= 1) return undefined

  /** loại trang đầu tiên */
  const first_type = list_page[0]?.page?.type

  /** nếu không có loại trang thì thôi */
  if (!first_type) return undefined

  /** kiểm tra xem tất cả trang có cùng loại không */
  const is_same_type = list_page.every(p => p?.page?.type === first_type)

  /** nếu cùng loại thì trả về thông tin nhóm */
  if (is_same_type) {
    return {
      type: first_type,
      pages: list_page,
    }
  }

  /** nếu cùng loại thì trả về thông tin nhóm */
  if (is_same_type) {
    return {
      type: first_type,
      pages: list_page,
    }
  }

  return undefined
})

/** tên nhóm tự chọn hiện tại */
const current_group_name = ref<string>()

/** nhóm trang được chọn thủ công (không thuộc nhóm nào) */
const adhoc_page_group = computed(() => {
  /** danh sách trang đã chọn */
  const list_page = values(pageStore.selected_page_list_info)

  /** điều kiện: > 1 trang, không phải platform group, không phải custom group, không phải tất cả nhóm */
  if (
    list_page.length > 1 &&
    !selected_platform_group.value &&
    !current_group_name.value &&
    !is_show_all_groups.value
  ) {
    return list_page
  }

  return undefined
})

/** lấy tên nhóm */
/** có hiển thị tất cả nhóm không */
const is_show_all_groups = computed(() => {
  /** Lấy danh sách thông tin các trang đã chọn */
  const SELECTED_PAGES = values(pageStore.selected_page_list_info)

  /** Nếu số lượng trang chọn nhỏ hơn hoặc bằng 1 thì không hiển thị tất cả nhóm */
  if (SELECTED_PAGES.length <= 1) return false

  /** Nếu đang chọn nhóm nền tảng thì không hiển thị tất cả nhóm */
  if (selected_platform_group.value) return false

  /** Nếu đang chọn nhóm tự chọn thì không hiển thị tất cả nhóm */
  if (current_group_name.value) return false

  /** ID tổ chức đang được chọn */
  const ORG_ID = orgStore.selected_org_id
  /** Nếu không có ID tổ chức thì trả về false */
  if (!ORG_ID) return false

  /** Nếu đang chọn một nhóm cụ thể trong tổ chức thì không phải là tất cả nhóm */
  if (orgStore.selected_org_group?.[ORG_ID]) return false

  /** Lấy danh sách tất cả các trang hiện có */
  const ALL_PAGES = values(pageStore.all_page_list)
  /** Lấy bản đồ ánh xạ giữa trang và tổ chức */
  const MAP_PAGE_ORG = pageStore.map_orgs?.map_page_org || {}

  /** Tính tổng số lượng trang thuộc tổ chức đang chọn */
  const TOTAL_ORG_PAGES_COUNT = ALL_PAGES.filter(p => {
    /** Lấy ID trang Facebook */
    const PAGE_ID = p?.page?.fb_page_id
    /** Nếu không có ID trang thì bỏ qua */
    if (!PAGE_ID) return false
    /** Kiểm tra trang có thuộc tổ chức hiện tại không */
    return MAP_PAGE_ORG[PAGE_ID] === ORG_ID
  }).length

  /** Kiểm tra nếu tổng số trang lớn hơn 0 và số lượng trang đã chọn bằng tổng số trang của tổ chức */
  return (
    TOTAL_ORG_PAGES_COUNT > 0 && SELECTED_PAGES.length === TOTAL_ORG_PAGES_COUNT
  )
})

/** lấy tên nhóm */
/** Lấy tên nhóm dựa trên tổ chức và nhóm đang chọn */
async function fetchGroupName() {
  /** Lấy ID tổ chức hiện tại từ store */
  const ORG_ID = orgStore.selected_org_id
  /** Lấy ID nhóm tương ứng với tổ chức hiện tại */
  const GROUP_ID = orgStore.selected_org_group?.[ORG_ID || '']

  /** Reset giá trị tên nhóm về mặc định */
  current_group_name.value = undefined

  /** Kiểm tra nếu thiếu thông tin tổ chức, nhóm hoặc đang chọn tất cả nhóm */
  if (!ORG_ID || !GROUP_ID || GROUP_ID === 'ALL') return

  try {
    /** Gọi API lấy danh sách nhóm của tổ chức */
    const GROUPS = await new BillingAppGroup().readGroup(ORG_ID)
    /** Tìm thông tin chi tiết của nhóm dựa trên ID nhóm */
    const GROUP = GROUPS?.find((g: any) => g.group_id === GROUP_ID)
    /** Cập nhật tên nhóm vào biến trạng thái */
    current_group_name.value = GROUP?.group_name
  } catch (ERROR) {
    /** Ghi log lỗi nếu quá trình lấy thông tin nhóm thất bại */
    console.error(ERROR)
  }
}

/** danh sách trang của nhóm hiện tại */
const current_group_pages = computed(() => {
  if (!current_group_name.value) return []
  return values(pageStore.selected_page_list_info)
})

/** danh sách bấ kỳ trang nào được chọn */
const all_selected_pages = computed(() => {
  return values(pageStore.selected_page_list_info)
})

/** theo dõi khi nhóm thay đổi */
watch(() => orgStore.selected_org_group, fetchGroupName, {
  deep: true,
  immediate: true,
})

/** danh sách nhãn của các trang đã chọn */
const tags = computed(() => {
  /** các nhãn lưu dưới dạng hash table */
  let tags: Record<string, ILabel> = {}

  /** lặp qua các trang được chọn để gộp các nhãn của các trang lại 1 danh sách */
  map(pageStore.selected_page_list_info, item => {
    tags = { ...tags, ...item.label_list }
  })

  return tags
})

/** danh sách các nhân sự của các trang đã chọn */
const staffs = computed(() => {
  /** các nhãn lưu dưới dạng hash table */
  let staffs: Record<string, StaffInfo> = {}

  /** lặp qua các trang được chọn để gộp các nhãn của các trang lại 1 danh sách */
  map(pageStore.selected_page_list_info, item => {
    staffs = { ...staffs, ...item.staff_list }
  })

  return staffs
})

function addContent(result: string[], content: string[], title: string) {
  /** nếu không có thì thôi */
  if (!content.length) return
  /** nếu có thì thêm vào kết quả */
  result.push(`${title}: ${content.join(', ')}`)
}

/** theo dõi giá trị ô tìm kiếm */
watch(() => search_conversation.value, onSearchConversation)

/** lắng nghe trạng thái của phím tắt */
watch(
  () => commonStore.keyboard_shortcut,
  value => {
    /** nếu không phải tìm kiếm thì bỏ qua */
    if (value !== 'search_conversation') return

    /** nếu chưa search thì bật chế độ search */
    if (!is_search.value) $main.toggleSearch()
    /** nếu đã có search rồi thi focus vào ô tìm kiếm */ else
      ref_search_conversation.value?.focus()

    /** clear data */
    commonStore.keyboard_shortcut = ''
  }
)

class Main {
  /** chuyển đổi tab đang kích hoạt */
  activeTab(tab: IActiveTab) {
    /** thay đổi cờ */
    conversationStore.option_filter_page_data.conversation_type = tab

    /** nếu tab là dạng bài biết thì thêm param lên url */
    $router.push({
      query: {
        ...$router.currentRoute.value.query,
        tab: tab === 'POST' ? 'POST' : undefined,
      },
    })
  }
  /** chuyển đổi trạng thái tìm kiếm */
  async toggleSearch() {
    /** nếu đang tìm kiếm và có giá trị ô tìm kiếm thì không cho đóng ô tìm kiếm */
    if (is_search.value && search_conversation.value) return

    /** toggle trạng thái tìm kiếm */
    is_search.value = !is_search.value

    /** nếu là mở ô tìm kiếm */
    if (is_search.value) {
      /** chờ render xong */
      await nextTick()

      /** focus vào ô tìm kiếm */
      ref_search_conversation.value?.focus()
    }
  }
}
const $main = new Main()

onMounted(() => {
  if (conversationStore.option_filter_page_data.search) {
    /** load giá trị search được lưu từ local vào biến khi load lại trang */
    search_conversation.value = conversationStore.option_filter_page_data.search
  }
})
</script>
