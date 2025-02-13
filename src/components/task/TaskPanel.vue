<template>
  <div class="header">Task Plan Library</div>
  <div class="plan-panel-wrapper">
    <a-table class="plan-table" :columns="columns" :data-source="plansData.data" row-key="job_id"
      :pagination="paginationProp" :scroll="{ x: '100%', y: 600 }" @change="refreshData">
      <!-- 执行时间 -->
      <template #duration="{ record }">
        <div>
          <div>{{ formatTaskTime(record.execute_time) }}</div>
          <div>{{ formatTaskTime(record.end_time) }}</div>
        </div>
      </template>
      <!-- 任务类型 -->
      <template #taskType="{ record }">
        <div>{{ formatTaskType(record) }}</div>
      </template>
      <!-- 失控动作 -->
      <template #lostAction="{ record }">
        <div>{{ formatLostAction(record) }}</div>
      </template>
      <!-- 状态 -->
      <template #status="{ record }">
        <div>
          <div class="flex-display flex-align-center">
            <span class="circle-icon" :style="{backgroundColor: formatTaskStatus(record).color}"></span>
            {{ formatTaskStatus(record).text }}
            <a-tooltip v-if="!!record.code" placement="bottom" arrow-point-at-center >
              <template #title>
              <div>{{ getCodeMessage(record.code) }}</div>
              </template>
              <exclamation-circle-outlined class="ml5" :style="{color: commonColor.WARN, fontSize: '16px' }"/>
            </a-tooltip>
          </div>
          <div v-if="record.status === TaskStatus.Carrying">
            <a-progress :percent="record.progress || 0" />
          </div>
        </div>
      </template>
      <!-- 操作 -->
      <template #action="{ record }">
        <span class="action-area">
          <a-popconfirm
            v-if="record.status === TaskStatus.Wait"
            title="Are you sure you want to delete flight task?"
            ok-text="Yes"
            cancel-text="No"
            @confirm="onDeleteTask(record.job_id)"
          >
            <a-button type="primary" size="small">Delete</a-button>
          </a-popconfirm>
        </span>
      </template>
    </a-table>
  </div>
</template>

<script setup lang="ts">
import { reactive, ref } from '@vue/reactivity'
import { message } from 'ant-design-vue'
import { TableState } from 'ant-design-vue/lib/table/interface'
import { onMounted } from 'vue'
import { IPage } from '/@/api/http/type'
import { deleteTask, getWaylineJobs, Task } from '/@/api/wayline'
import { useMyStore } from '/@/store'
import { ELocalStorageKey } from '/@/types/enums'
import { useFormatTask } from './use-format-task'
import { TaskStatus, TaskProgressInfo, TaskProgressStatus, TaskProgressWsStatusMap } from '/@/types/task'
import { useTaskProgressEvent } from './use-task-progress-event'
import { getErrorMessage } from '/@/utils/error-code/index'
import { commonColor } from '/@/utils/color'
import { ExclamationCircleOutlined } from '@ant-design/icons-vue'

const store = useMyStore()
const workspaceId = localStorage.getItem(ELocalStorageKey.WorkspaceId)!

const body: IPage = {
  page: 1,
  total: 0,
  page_size: 50
}
const paginationProp = reactive({
  pageSizeOptions: ['20', '50', '100'],
  showQuickJumper: true,
  showSizeChanger: true,
  pageSize: 50,
  current: 1,
  total: 0
})

const columns = [
  {
    title: 'Planned/Actual Time',
    dataIndex: 'duration',
    width: 180,
    slots: { customRender: 'duration' },
  },
  {
    title: 'Plan Name',
    dataIndex: 'job_name',
    width: 150,
    ellipsis: true
  },
  {
    title: 'Type',
    dataIndex: 'taskType',
    width: 120,
    slots: { customRender: 'taskType' },
  },
  {
    title: 'Flight Route Name',
    dataIndex: 'file_name',
    width: 150,
    ellipsis: true
  },
  {
    title: 'Dock Name',
    dataIndex: 'dock_name',
    width: 120,
    ellipsis: true
  },
  {
    title: 'RTH Altitude Relative to Dock (m)',
    dataIndex: 'rth_altitude',
    width: 120,
    ellipsis: true
  },
  {
    title: 'Lost Action',
    dataIndex: 'out_of_control_action',
    width: 120,
    slots: { customRender: 'lostAction' },
  },
  {
    title: 'Creator',
    dataIndex: 'username',
    width: 120,
  },
  {
    title: 'Status',
    key: 'status',
    width: 200,
    slots: { customRender: 'status' }
  },
  {
    title: 'Action',
    width: 120,
    slots: { customRender: 'action' }
  }
]
type Pagination = TableState['pagination']

const plansData = reactive({
  data: [] as Task[]
})

const { formatTaskType, formatTaskTime, formatLostAction, formatTaskStatus } = useFormatTask()

// 设备任务执行进度更新
function onTaskProgressWs (data: TaskProgressInfo) {
  const { bid, output } = data
  if (output) {
    const { status, progress } = output || {}
    const taskItem = plansData.data.find(task => task.job_id === bid)
    if (!taskItem) return
    if (status) {
      taskItem.status = TaskProgressWsStatusMap[status]
      // 执行中，更新进度
      if (status === TaskProgressStatus.Sent || status === TaskProgressStatus.inProgress) {
        taskItem.progress = progress?.percent || 0
      } else if ([TaskProgressStatus.Rejected, TaskProgressStatus.Canceled, TaskProgressStatus.Timeout, TaskProgressStatus.Failed, TaskProgressStatus.OK].includes(status)) {
        getPlans()
      }
    }
  }
}

function getCodeMessage (code: number) {
  return getErrorMessage(code) + `（code: ${code}）`
}

useTaskProgressEvent(onTaskProgressWs)

onMounted(() => {
  getPlans()
})

function getPlans () {
  getWaylineJobs(workspaceId, body).then(res => {
    if (res.code !== 0) {
      return
    }
    plansData.data = res.data.list
    paginationProp.total = res.data.pagination.total
    paginationProp.current = res.data.pagination.page
  })
}

function refreshData (page: Pagination) {
  body.page = page?.current!
  body.page_size = page?.pageSize!
  getPlans()
}

// 删除任务
async function onDeleteTask (jobId: string) {
  const { code } = await deleteTask(workspaceId, {
    job_id: jobId
  })
  if (code === 0) {
    message.success('Deleted successfully')
    getPlans()
  }
}
</script>

<style lang="scss" scoped>
.plan-panel-wrapper {
  width: 100%;
  padding: 16px;
  .plan-table {
    background: #fff;
    margin-top: 10px;
  }
  .action-area {
    color: $primary;
    cursor: pointer;
  }

  .circle-icon {
    display: inline-block;
    width: 12px;
    height: 12px;
    margin-right: 3px;
    border-radius: 50%;
    vertical-align: middle;
    flex-shrink: 0;
  }
}
.header {
  width: 100%;
  height: 60px;
  background: #fff;
  padding: 16px;
  font-size: 20px;
  font-weight: bold;
  text-align: start;
  color: #000;
}
</style>
