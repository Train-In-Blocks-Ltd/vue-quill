<style>
  div.ql-container {
    font-size: 16px
  }
  .ui.attached.segment.ql-container.ql-snow {
    border: none
  }
  .ql-toolbar.ql-snow {
    padding: 0;
    border: none
  }
  .ql-toolbar.ql-snow button {
    padding-left: 0
  }
  .ql-editor {
    grid-area: body;
    color: #282828;
    overflow-y: auto;
    padding: 0;
    transition: all 1s
  }
  .ql-snow .ql-editor img {
    max-width: 60%
  }
  div.ql-editor h1 {
    margin: 1.072rem 0
  }
  div.ql-editor h2 {
    margin: 1.245rem 0
  }
  div.ql-editor p {
    margin: 1rem 0
  }
  .ql-editor.ql-blank:before {
    margin: 1rem 0;
    left: 0
  }
  @media (max-width: 768px) {
    .ql-toolbar.ql-snow {
      border: none;
      padding: .4rem .6rem .4rem .6rem;
      position: fixed;
      top: 25%;
      right: 0;
      border-radius: 3px;
      background-color: #F4F4F4;
      z-index: 2
    }
    .ql-toolbar.ql-snow .ql-formats {
      display: grid;
      margin: 0
    }
    .ql-snow.ql-toolbar button, .ql-snow .ql-toolbar button {
      margin: .2rem 0
    }
    .ql-editor img {
      max-width: 100%
    }
  }
</style>

<template>
  <div>
    <div class="ui attached segment" ref="quill" @click.prevent="focusEditor"></div>
  </div>
</template>

<script>
  import defaultsDeep from 'lodash.defaultsdeep'
  import Quill from 'quill'
  import GrammarlyInline from './formats/GrammarlyInline'
  import ImageCompress from 'quill-image-compress'
  import { ImageDrop } from 'quill-image-drop-module'
  import 'quill/dist/quill.snow.css'

  export default {
      model: {
          prop: 'content',
      },

      props: {
          content: {},

          formats: {
              type: Array,
              default() {
                  return []
              },
          },

          keyBindings: {
              type: Array,
              default() {
                  return []
              },
          },

          output: {
              default : 'delta'
          },

          bus: {
              default: false,
          },

          config: {
              default() {
                  return {}
              },
          },
      },

      data() {
          return {
              editor: {},
              placeholder: 'Type away...',
              defaultConfig: {
                  modules: {
                    imageDrop: true,
                    imageCompress: {
                      quality: 0.7,
                      maxWidth: 500,
                      maxHeight: 500,
                      imageType: 'image/jpeg',
                      debug: false
                    },
                    clipboard: {
                      matchVisual: false
                    },
                    toolbar: [
                        ['bold', 'italic', 'underline', {'list': 'ordered'}, {'list': 'bullet'}, { 'list': 'check' }, 'link', 'image']
                    ],
                  },
                  theme: 'snow',
              },
          }
      },

      mounted() {
          if (this.keyBindings.length) {
              this.defaultConfig.modules.keyboard = {
                  bindings: this.keyBindings.map((binding) => {
                      if (binding.remove) return false
                      return {
                          key: binding.key,
                          metaKey: true,
                          handler: binding.method.bind(this),
                      }
                  })
              }
          }

          if (this.config.modules && this.config.modules.toolbar) {
              this.defaultConfig.modules.toolbar = []
          }

          Quill.register(GrammarlyInline)
          Quill.register('modules/imageCompress', ImageCompress)
          Quill.register('modules/quillMobileView', mobileView.module)
          Quill.register('modules/imageDrop', ImageDrop)

          this.editor = new Quill(this.$refs.quill, defaultsDeep(this.config, this.defaultConfig))

          if (this.content && this.content !== '') {
            if (this.output != 'delta') {
                this.editor.pasteHTML(this.content)
            } else {
                this.editor.setContents(this.content)
            }
          }

          this.editor.on('text-change', (delta, source) => {
              this.$emit('text-change', this.editor, delta, source)
              this.$emit('input', this.output != 'delta' ? this.editor.root.innerHTML : this.editor.getContents())
          })

          this.editor.on('selection-change', (range) => {
              this.$emit('selection-change', this.editor, range)
          })

          if (this.bus) {
              this.bus.$on('focus-editor', () => this.focusEditor())
              this.bus.$on('set-content', (content) => this.editor.setContents(content))
              this.bus.$on('set-html', (html) => {
                  if (!html || html === '') return

                  this.editor.root.innerHTML = html
              })
          }

          this.$on('focus-editor', () => this.focusEditor())
          this.$on('set-content', (content) => this.editor.setContents(content))
          this.$on('set-html', (html) => {
              if (!html || html === '') return

              this.editor.root.innerHTML = html
          })

          this.$nextTick(() => {
              const selectors = ['button', '.ql-picker-label', '.ql-picker-item']
              const toolbar = this.$el.querySelector('.ql-toolbar')
              selectors.forEach((selector) => {
                  toolbar.querySelectorAll(selector).forEach((element) => {
                      element.tabIndex = -1
                  })
              })
          })
      },

      methods: {
          focusEditor(e) {
              if (e && e.srcElement) {
                  let classList = e.srcElement.classList,
                      isSegment = false

                  classList.forEach((className) => {
                      if (className === 'segment') {
                          isSegment = true
                          return
                      }
                  })

                  if (!isSegment) return
              }

              this.editor.focus()
              this.editor.setSelection(this.editor.getLength()-1, this.editor.getLength())
          }
      },

      beforeDestroy() {
          if (this.bus) {
              this.bus.$off('focus-editor')
              this.bus.$off('set-content')
              this.bus.$off('set-html')
          }
      },
  }
</script>
