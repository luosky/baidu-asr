<template lang='pug'>
a-space(direction="vertical" fill :size="10")
  a-form.form(:model="form")
    a-space(direction="vertical" :size="10")
      a-input(v-model="form.ocrAPI" placeholder="请先填写OCR API，配置方式见下方说明")
        template(#prefix)
          icon-robot
      .info
        icon-info-circle
        a(href="https://yellowduck.feishu.cn/docx/Q89SdbfZyoI50RxiwxlcTtP8nkb?from=from_copylink" target="_blank") 查看OCR接口配置文档
      a-select(v-if="form.ocrAPI" v-model="form.attachment" placeholder="选择图片列")
        template(#prefix)
          icon-file-image
        a-option(v-for="table in attachmentField" :key="table.id" :value="table.id") {{ table.name }}
      a-select(v-if="!!form.attachment" v-model="form.text" placeholder="选择文本列")
        template(#prefix)
          icon-file
        a-option(v-for="view in textField" :key="view.id" :value="view.id") {{ view.name }}
  a-space(size="large")
    a-statistic(
      v-if="form.attachment"
      animation
      title="图片数"
      show-group-separator
      :value="total.length")
    a-statistic(
      v-if="form.attachment && form.text"
      animation
      title="未识别图片数"
      show-group-separator
      :value="todo.length")
  a-button(
    type="primary"
    :disabled="!form.text"
    :loading="loading"
    @click="run") 开始识别
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
  const selection = ref({})
  const fieldMetaList = ref([])

  const form = reactive({
    attachment: '',
    text: '',
    ocrAPI: useStorage('ocrAPI', ''),
  })

  const attachmentField = computed(() => fieldMetaList.value.filter(meta => meta.type === FieldType.Attachment))
  const textField = computed(() => fieldMetaList.value.filter(meta => meta.type === FieldType.Text))
  const total = computed(() => records.value.filter(record => record.fields[form.attachment]))
  const todo = computed(() => records.value.filter(record => record.fields[form.attachment] && !record.fields[form.text]))

  // methods
  const imageToText = async url => {
    const res = await axios.post(form.ocrAPI, {
      url,
    })

    return [res.data.text]
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

  watch(
    () => loading.value && !todo.value.length,
    async value => {
      if (value) {
        loading.value = false
      }
    }
  )

  const run = async () => {
    await fetchRecords()
    loading.value = true
    todo.value.forEach(async record => {
      let attachmentToken = record.fields[form.attachment][0].token
      let attachmentURL = await table.getAttachmentUrl(attachmentToken)

      const texts = (await imageToText(attachmentURL)).map(text => ({
        type: IOpenSegmentType.Text,
        text: text + '\n',
      }))

      table.setCellValue(form.text, record.recordId, texts)
    })
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