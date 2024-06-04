<template lang='pug'>
a-space(direction="vertical" fill :size="10")
  a-form.form(:model="form")
    a-space(direction="vertical" :size="10")
      a-input(v-model="form.ocrAPI" placeholder="请先填写ASR API")
        template(#prefix)
          icon-robot
      .info
      a-select(v-if="form.ocrAPI" v-model="form.attachment" placeholder="选择录音文件列")
        template(#prefix)
          icon-file-audio
        a-option(v-for="table in attachmentField" :key="table.id" :value="table.id") {{ table.name }}
      a-select(v-if="!!form.attachment" v-model="form.text" placeholder="选择任务 id列")
        template(#prefix)
          icon-file
        a-option(v-for="view in textField" :key="view.id" :value="view.id") {{ view.name }}
  a-space(size="large")
    a-statistic(
      v-if="form.attachment"
      animation
      title="录音数"
      show-group-separator
      :value="total.length")
    a-statistic(
      v-if="form.attachment && form.text"
      animation
      title="待提交录音数"
      show-group-separator
      :value="todo.length")
  a-button(
    type="primary"
    :disabled="!form.text"
    :loading="loading"
    @click="run") 开始提交任务
  a-input(v-model="jobForm.jobAPI" placeholder="请先填写获取 job 的 api")
        template(#prefix)
          icon-robot
  a-select(v-if="!!jobForm.jobAPI" v-model="jobForm.result" placeholder="选择解析结果列")
        template(#prefix)
          icon-file
        a-option(v-for="view in textField" :key="view.id" :value="view.id") {{ view.name }}
  a-space(size="large")
    a-statistic(
      v-if="form.text"
      animation
      title="job 数"
      show-group-separator
      :value="jobTotal.length")
    a-statistic(
      v-if="form.text && jobForm.result"
      animation
      title="job 完成数"
      show-group-separator
      :value="jobDone.length")
    a-statistic(
      v-if="form.text && jobForm.result"
      animation
      title="待识别录音数"
      show-group-separator
      :value="jobTodo.length")
  a-button(
    type="primary"
    :disabled="!jobForm.result"
    :loading="fetching"
    @click="fetch_result") 开始获取任务结果
</template>
<script setup>
  import { bitable, FieldType, IOpenSegmentType } from '@lark-base-open/js-sdk'
  import { useStorage } from '@vueuse/core'
  import axios from 'axios'
  import { watch } from 'vue'

  // data
  let table = {}
  const pageSize = 5000
  const records = ref([])
  const loading = ref(false)
  const fetching = ref(false)
  const selection = ref({})
  const fieldMetaList = ref([])

  const form = reactive({
    attachment: useStorage('asrAttachment', ''),
    text: useStorage('asrJobID', ''),
    ocrAPI: useStorage('asrAPI', ''),
  })


  const jobForm = reactive({
    result: useStorage('asrResult', ''),
    jobAPI: useStorage('jobAPI', ''),
  })

  const attachmentField = computed(() => fieldMetaList.value.filter(meta => meta.type === FieldType.Attachment))
  const textField = computed(() => fieldMetaList.value.filter(meta => meta.type === FieldType.Text))
  
  const total = computed(() => records.value.filter(record => record.fields[form.attachment]))
  const todo = computed(() => records.value.filter(record => record.fields[form.attachment] && !record.fields[form.text]))
  const jobDone = computed(() => records.value.filter(record => record.fields[form.text] && record.fields[jobForm.result] && record.fields[jobForm.result][0].text != "Running" && record.fields[jobForm.result][0].text != "Failure"))
  const jobTotal = computed(() => records.value.filter(record => record.fields[form.text] && !record.fields[form.text][0].text.includes("创建任务失败")))
  const jobTodo = computed(() => records.value.filter(record => (record.fields[form.text] && !record.fields[form.text][0].text.includes("创建任务失败")) && (!record.fields[jobForm.result] || record.fields[jobForm.result][0].text == "Running")))

  
  // methods
  const audioFileToText = async url => {
    const res = await axios.post(form.ocrAPI, {
      url,
    })

    return [res.data.text]
  }

  const jobIDToText = async job => {
    const res = await axios.post(jobForm.jobAPI, {
      job,
    })
    const task = res.data

    if (task.status == "Success") {
      console.log(`task success, ${task.task_id}`)
      return task.result
    } else {
      console.log(`task ${task.status}, ${task.task_id}`)
      return [task.status]
    }
  }

  const fetchRecords = async () => {
    let hasMore = true
    let pageToken = ''
    let tempRecords = []

    while (hasMore) {
      const res = await table.getRecords({ pageToken, pageSize })

      hasMore = res.hasMore
      pageToken = res.pageToken
      tempRecords = tempRecords.concat(res.records)
    }
    records.value = tempRecords
  }

  watch(form, async () => {
    fetchRecords()
  })

  watch(jobForm, async () => {
    fetchRecords()
  })

  watch(
    () => loading.value && !todo.value.length,
    async value => {
      if (value) {
        loading.value = false
      }
    }
  )

  watch(
    () => fetching.value && !jobTodo.value.length && !(jobTotal.value.length - jobTodo.value.length - jobDone.value.length) ,
    async value => {
      if (value) {
        fetching.value = false
      }
    }
  )
const concurrency = 2
const runConcurrent = async (concurrency, items, callback) => {
console.log(`items size: ${items.length}, concurrency: ${concurrency}`)
  const batches = [];

  while (items.length > 0) {  
    const batch = items.splice(0, concurrency);
    console.log(`batch size: ${batch.length}`)
    batches.push(batch);
  }
  console.log(`batches size : ${batches.length}`)

  // 逐个执行每个小批次
  var batch_no = 0
  for (const batch of batches) {
    batch_no += 1
    console.log(`executing batch ${batch_no}/${batches.length}...`)
    await Promise.all(batch.map(async item => {
      await callback(item)
    }))
  }

}

const run = async () => {
  console.log("running1...")
  await fetchRecords()
  loading.value = true
  var summited = 0
  const records = todo.value
  const getAttachmentUrlAndSubmitJob = async record => {
    summited += 1
    let attachmentToken = record.fields[form.attachment][0].token
    let attachmentURL = await table.getAttachmentUrl(attachmentToken)
    console.log(`#${summited}, attachmentURL : ${attachmentURL}`)
    const texts = (await audioFileToText(attachmentURL)).map(text => ({
      type: IOpenSegmentType.Text,
      text: text,
    }))

    table.setCellValue(form.text, record.recordId, texts)
  }

  console.log(`records count : ${records.length}`)

  await runConcurrent(concurrency, records, getAttachmentUrlAndSubmitJob)
/*
  const batches = [];

  while (records.length > 0) {  
    const batch = records.splice(0, concurrency);
    console.log(`batch size: ${batch.length}`)
    batches.push(batch);
  }
  console.log(`batches count : ${batches.length}`)

  console.log("running3...")
  // 逐个执行每个小批次
  for (const batch of batches) {
    console.log(`executing batch...`)
    await Promise.all(batch.map(async record => {
      await getAttachmentUrlAndSubmitJob(record)
    }))
  }
*/

  /*
  todo.value.forEach(async record => {
    let attachmentToken = record.fields[form.attachment][0].token
    let attachmentURL = await table.getAttachmentUrl(attachmentToken)

    const texts = (await audioFileToText(attachmentURL)).map(text => ({
      type: IOpenSegmentType.Text,
      text: text,
    }))

    table.setCellValue(form.text, record.recordId, texts)
  })
  */
  // loading.value = false
}

const fetch_result = async () => {
  console.log(`fetching result..., form.text : ${form.text}`)
  await fetchRecords()
  fetching.value = true 
  var doneFetchingCount = 0
  const needFetchingCount = jobTodo.value.length
  const fetch_record_result = async record => {
    console.log(`record : ${JSON.stringify(record)}`)
    const jobIDStr = record.fields[form.text][0].text

    const texts = (await jobIDToText(jobIDStr)).map(text => ({
      type: IOpenSegmentType.Text,
      text: text,
    }))
    
    table.setCellValue(jobForm.result, record.recordId, texts)
    doneFetchingCount += 1
    if (doneFetchingCount == needFetchingCount) {
      fetching.value = false
    }
  }
  await runConcurrent(concurrency, jobTodo.value, fetch_record_result)
  
  
  }
  // lifecycle
  onMounted(async () => {
    selection.value = await bitable.base.getSelection()
    table = await bitable.base.getTableById(selection.value.tableId)
    table.onRecordModify((recordId, filedIds) => {
      fetchRecords()
    })
    const view = await table.getViewById(selection.value.viewId)
    fieldMetaList.value = await view.getFieldMetaList()
    fetchRecords()
  })
</script>
<style lang='stylus' rel='stylesheet/stylus' scoped>
  .form {
    padding: 10px 0;
  }

  .info {
    display: flex;
    align-items: center;

    svg {
      margin-right: 4px;
    }

    a {
      color: var(--color-text);
    }
  }
</style>