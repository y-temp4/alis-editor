<template lang="html">
  <div id="ALISEditor">
    <div class="container" id="editor"></div>
    <InsertButton
      :articleId="articleId"
      :editor="editor"
      v-if="insertButton.isVisible"
      :style="{
        left: `calc(50% - 400px)`,
        top: `${insertButton.posY}px`
      }"
    />
  </div>
</template>

<script>
import BalloonEditor from '@ckeditor/ckeditor5-editor-balloon/src/ballooneditor'
import BoldPlugin from '@ckeditor/ckeditor5-basic-styles/src/bold'
import ItalicPlugin from '@ckeditor/ckeditor5-basic-styles/src/italic'
import LinkPlugin from '@ckeditor/ckeditor5-link/src/link'
import Paragraph from '@ckeditor/ckeditor5-paragraph/src/paragraph'
import Heading from '@/plugins/heading'
import HeadingButtonsUI from '@ckeditor/ckeditor5-heading/src/headingbuttonsui'
import BlockQuote from '@ckeditor/ckeditor5-block-quote/src/blockquote'
import EssentialsPlugin from '@ckeditor/ckeditor5-essentials/src/essentials'
import Image from '@ckeditor/ckeditor5-image/src/image'
import ImageToolbar from '@ckeditor/ckeditor5-image/src/imagetoolbar'
import ImageCaption from '@ckeditor/ckeditor5-image/src/imagecaption'
import ImageStyle from '@ckeditor/ckeditor5-image/src/imagestyle'
import ImageUpload from '@ckeditor/ckeditor5-image/src/imageupload'
import InsertButton from './components/InsertButton'
import CustomUploadAdapterPlugin from '@/plugins/CustomUploadAdapterPlugin'
import Autosave from '@ckeditor/ckeditor5-autosave/src/autosave'
import Emptyness from 'ckeditor5-emptyness/src/emptyness'
import MediaEmbed from '@ckeditor/ckeditor5-media-embed/src/mediaembed'
import saveData from '@/utils/Save'
import iconHeading2 from '@/assets/icons/heading2.svg'
import iconHeading3 from '@/assets/icons/heading3.svg'
import { IFRAMELY_API_ENDPOINT } from '@/utils/constant'

export default {
  props: {
    articleId: {
      type: String
    },
    clientId: {
      type: String
    },
    functions: {
      type: Object
    },
    editorContent: {
      type: String,
      default: null
    },
    iframelyApiKey: {
      type: String
    },
    domain: {
      type: String
    }
  },
  components: {
    InsertButton
  },
  data() {
    return {
      editor: null,
      insertButton: {
        isVisible: false,
        posY: 0
      }
    }
  },
  mounted() {
    // プラスボタンの挙動制御
    document.addEventListener('selectionchange', this.controlButton)
    // propsを変数にset
    const { articleId, clientId, functions } = this
    BalloonEditor.create(document.querySelector('#editor'), {
      extraPlugins: [CustomUploadAdapterPlugin.bind(null, articleId, clientId, functions)],
      plugins: [
        EssentialsPlugin,
        BoldPlugin,
        ItalicPlugin,
        LinkPlugin,
        Heading,
        HeadingButtonsUI,
        Paragraph,
        BlockQuote,
        Image,
        ImageToolbar,
        ImageCaption,
        ImageStyle,
        ImageUpload,
        Autosave,
        Emptyness,
        MediaEmbed
      ],
      toolbar: ['heading2', 'heading3', 'blockQuote', 'bold', 'italic', 'link'],
      autosave: {
        save(editor) {
          return saveData(editor.getData(), articleId, clientId, functions)
        }
      },
      heading: {
        options: [
          { model: 'paragraph', title: 'Paragraph', class: 'ck-heading_paragraph' },
          {
            model: 'heading2',
            view: 'h2',
            title: 'Heading 2',
            class: 'ck-heading_heading2',
            icon: iconHeading2
          },
          {
            model: 'heading3',
            view: 'h3',
            title: 'Heading 3',
            class: 'ck-heading_heading3',
            icon: iconHeading3
          }
        ]
      },
      image: {
        toolbar: ['imageStyle:alignLeft', 'imageStyle:full', 'imageStyle:alignRight'],
        styles: [
          'full',
          { name: 'alignLeft', title: '左寄せ画像' },
          { name: 'alignRight', title: '右寄せ画像' }
        ]
      },
      mediaEmbed: {
        previewsInData: false,
        providers: [
          {
            name: 'twitter',
            url: /^twitter\.com/,
            html: (match) => {
              const path = match['input']
              const isTweet = path.split('/')[2] === 'status'
              const iframeUrl = `${IFRAMELY_API_ENDPOINT}?app=1&api_key=${
                this.iframelyApiKey
              }&url=${encodeURIComponent(path)}`
              if (isTweet) {
                return `<div class="iframely-embed">
                     <div class="iframely-responsive">
                       <iframe src="${iframeUrl}" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen>
                       </iframe>
                     </div>
                   </div>`
              }
              const userName = path.split('/')[1]
              return `<div class="iframe-twitter">
                   <iframe src="https://${this.domain}/media_embed/twitter_profile/${userName}"
                   frameborder="0" allow="autoplay; encrypted-media" allowfullscreen class="twitter-content-area">
                   </iframe>
                 </div>`
            }
          },
          {
            name: 'facebook',
            url: /^facebook\.com/,
            html: (match) => {
              const path = 'https://' + match['input']
              const iframeUrl = `${IFRAMELY_API_ENDPOINT}?app=1&api_key=${
                this.iframelyApiKey
              }&url=${encodeURIComponent(path)}`
              return `<div class="iframely-embed">
                   <div class="iframely-responsive">
                     <iframe src="${iframeUrl}" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen>
                     </iframe>
                   </div>
                 </div>`
            }
          },
          {
            name: 'youtube',
            url: [
              /^youtube\.com\/watch\?v=([\w-]+)/,
              /^youtube\.com\/v\/([\w-]+)/,
              /^youtube\.com\/embed\/([\w-]+)/,
              /^youtu\.be\/([\w-]+)/
            ],
            html: (match) => {
              const id = match[1]
              return `<div class="iframe-youtube">
                   <iframe src="https://www.youtube.com/embed/${id}" class="youtube-content-area"
                   frameborder="0" allow="autoplay; encrypted-media" allowfullscreen>
                   </iframe>
                 </div>`
            }
          },
          {
            name: 'gist',
            url: /^gist\.github\.com/,
            html: (match) => {
              const path = 'https://' + match['input']
              const iframeUrl = `${IFRAMELY_API_ENDPOINT}?app=1&api_key=${
                this.iframelyApiKey
              }&url=${encodeURIComponent(path)}`
              return `<div class="iframely-embed">
                   <div class="iframely-responsive">
                     <iframe src="${iframeUrl}" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen>
                     </iframe>
                   </div>
                 </div>`
            }
          },
          {
            name: 'instagram',
            url: /^www\.instagram\.com\/p\/(\w+)/,
            html: (match) => {
              const path = 'https://' + match['input']
              const iframeUrl = `${IFRAMELY_API_ENDPOINT}?app=1&api_key=${
                this.iframelyApiKey
              }&url=${encodeURIComponent(path)}`
              return `<div class="iframely-embed">
                   <div class="iframely-responsive">
                     <iframe src="${iframeUrl}" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen>
                     </iframe>
                   </div>
                 </div>`
            }
          },
          {
            name: 'any',
            url: /.+/,
            html: (match) => {
              const path = match[0]
              return `<div class="iframe-any">
                   <iframe src="https://${this.domain}/media_embed/any?url=${encodeURIComponent(
                path
              )}"
                   frameborder="0" allow="autoplay; encrypted-media" allowfullscreen class="any-content-area">
                   </iframe>
                 </div>`
            }
          }
        ]
      }
    }).then((editor) => {
      this.editor = editor
      if (this.editorContent !== null) {
        editor.setData(this.editorContent)
      }
    })
  },
  methods: {
    controlButton() {
      const selection = window.getSelection()
      const target = selection.anchorNode
      if (target === null) {
        this.insertButton.isVisible = false
        return
      }
      if (target.nodeName !== 'P') {
        this.insertButton.isVisible = false
        return
      }
      if (target.textContent === '') {
        const rect = target.getBoundingClientRect()
        this.insertButton.posY = rect.top - 13 + window.pageYOffset
        this.insertButton.isVisible = true
      } else {
        this.insertButton.isVisible = false
      }
    }
  },
  beforeDestroy() {
    document.removeEventListener('selectionchange', this.controlButton)
  }
}
</script>

<style lang="scss">
.iframe-twitter {
  position: relative;
  padding-bottom: 140px;

  .twitter-content-area {
    position: absolute;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
  }
}
.iframe-any {
  position: relative;
  padding-bottom: 140px;

  .any-content-area {
    position: absolute;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
  }
}

.iframe-youtube {
  position: relative;
  padding-bottom: 100%;
  height: 0;
  padding-bottom: 56.2493%;
  .youtube-content-area {
    position: absolute;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
  }
}
</style>
