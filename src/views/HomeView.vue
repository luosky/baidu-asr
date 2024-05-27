<template lang='pug'>
a-space(direction="vertical" fill :size="10")
  a-spin(:loading="tableLoading" :tip="$t('table.loading')")
    a-form.form(:model="form")
      a-space(direction="vertical" :size="10")
        a-input(v-model="form.ocrAPI" :placeholder="$t('ocr.placeholder')")
          template(#prefix)
            icon-robot
        .info
          icon-info-circle
          a(href="https://yellowduck.feishu.cn/docx/Q89SdbfZyoI50RxiwxlcTtP8nkb?from=from_copylink" target="_blank") {{ $t('ocr.notice') }}
          a-button.check(
            size="mini"
            type="primary"
            :loading="isChecking"
            @click="checkAPI") {{ isChecking ? $t('checking') : $t('check') }}
        a-select(v-if="form.ocrAPI" v-model="form.attachment" :placeholder="$t('attachment.placeholder')")
          template(#prefix)
            icon-file-image
          a-option(v-for="table in attachmentField" :key="table.id" :value="table.id") {{ table.name }}
        a-select(v-if="!!form.attachment" v-model="form.text" :placeholder="$t('text.placeholder')")
          template(#prefix)
            icon-file
          a-option(v-for="view in textField" :key="view.id" :value="view.id") {{ view.name }}
  a-space(size="large")
    a-statistic(
      v-if="form.attachment"
      animation
      show-group-separator
      :title="$t('image.total')"
      :value="total.length")
    a-statistic(
      v-if="form.attachment && form.text"
      animation
      show-group-separator
      :title="$t('image.todo')"
      :value="todo.length")
  a-button(
    type="primary"
    :disabled="!form.text"
    :loading="OCRLoading"
    @click="run") {{ $t('start') }}
</template>
<script setup>
  import { Message } from '@arco-design/web-vue'
  import { bitable, FieldType, IOpenSegmentType } from '@lark-base-open/js-sdk'
  import { useStorage } from '@vueuse/core'
  import axios, { formToJSON } from 'axios'
  import { watch } from 'vue'
  // import i18n from '@/locale'

  // data
  let table = {}
  // const { t } = i18n.global
  const pageSize = 5000

  const OCRLoading = ref(false)
  const tableLoading = ref(false)

  const records = ref([])
  const selection = ref({})
  const fieldMetaList = ref([])

  const isChecking = ref(false)

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

    return res.data.text
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
    () => OCRLoading.value && !todo.value.length,
    async value => {
      if (value) {
        OCRLoading.value = false
      }
    }
  )

  const checkAPI = async () => {
    isChecking.value = true
    if (!form.ocrAPI) {
      Message.error(t('error.inputAPI'))
      isChecking.value = false
      return
    }
    const url = 'https://yupi-picture-1256524210.cos.ap-shanghai.myqcloud.com/wonderful/1699937792355.png'
    const res = await imageToText(url)
    if (res[0] === '这张图片用来测试API接口是否正常工作') {
      Message.success(t('success.ocrAPI'))
    } else {
      Message.error(t('error.ocrAPI'))
    }
    isChecking.value = false
  }

  const run = async () => {
    await fetchRecords()
    OCRLoading.value = true
    todo.value.forEach(async record => {
      let attachmentToken = record.fields[form.attachment][0].token
      let attachmentURL = await table.getAttachmentUrl(attachmentToken)

      if (!attachmentURL) {
        Message.error(t('error.wrongArrachmentURL'))
      }

      const texts = (await imageToText(attachmentURL)).map(text => ({
        type: IOpenSegmentType.Text,
        text: text + '\n',
      }))

      table.setCellValue(form.text, record.recordId, texts)
    })
  }
  // lifecycle
  const reloadTable = async () => {
    form.attachment = ''
    form.text = ''

    tableLoading.value = true
    selection.value = await bitable.base.getSelection()
    table = await bitable.base.getTableById(selection.value.tableId)

    table.onRecordModify((recordId, filedIds) => {
      fetchRecords()
    })
    const view = await table.getViewById(selection.value.viewId)
    fieldMetaList.value = await view.getFieldMetaList()
    tableLoading.value = false
  }

  onMounted(async () => {
    bitable.base.onSelectionChange(() => {
      reloadTable()
    })
    reloadTable()
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

  :deep(.arco-spin) {
    width: 100%;
  }

  .check {
    margin-left: 5px;
  }
</style>