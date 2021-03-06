<template>
  <div>
    <div v-if="contest">
      <v-container class="pa-0" fluid>
        <v-card class="mx-auto" max-width="800px" :loading="!problem">
          <template v-if="problem">
            <v-card-title class="mb-3">
              <p style="width:100%">
                {{ problem.position }}: {{ problem.name }}
              </p>
              <DifficultyChip :difficulty="problem.difficulty" />
            </v-card-title>
            <v-card-text style="color:inherit">
              <section>
                <h3>問題文</h3>
                <div class="statement" v-html="$md.render(problem.statement)" />
              </section>
              <section>
                <h3>制約</h3>
                <div
                  class="statement"
                  v-html="$md.render(problem.constraints)"
                />
              </section>
              <section>
                <h3>入力</h3>
                <div
                  class="statement"
                  v-html="$md.render(problem.inputFormat)"
                />
              </section>
              <section>
                <h3>出力</h3>
                <div
                  class="statement"
                  v-html="$md.render(problem.outputFormat)"
                />
              </section>
              <section v-for="(sample, index) in problem.samples" :key="index">
                <div class="sample__head">
                  <div class="sample__head__title">入力例 {{ index + 1 }}</div>
                  <v-btn
                    color="blue lighten-4"
                    small
                    @click="copy(sample.input)"
                    >コピー</v-btn
                  >
                </div>
                <div class="statement">
                  <code class="sample__code" v-html="sample.input" />
                </div>
                <div class="sample__head">
                  <div class="sample__head__title">出力例 {{ index + 1 }}</div>
                  <v-btn
                    color="blue lighten-4"
                    small
                    @click="copy(sample.output)"
                    >コピー</v-btn
                  >
                </div>
                <div class="statement">
                  <code class="sample__code" v-html="sample.output" />
                </div>
                <div
                  v-if="sample.explanation"
                  class="statement"
                  v-html="$md.render(sample.explanation)"
                />
              </section>
              <section>
                <div class="submit__head">
                  <div class="submit__head__title">提出</div>
                  <v-autocomplete
                    v-model="language"
                    class="submit__head__select"
                    :items="selectableLanguages"
                    label="提出する言語"
                    dense
                    hide-details
                    outlined
                  />
                </div>
                <Editor
                  ref="editor"
                  class="submit__editor"
                  :language="language"
                />
                <v-btn
                  color="primary"
                  class="mt-2"
                  :disabled="!language || submitted"
                  @click="submit"
                >
                  提出する
                </v-btn>
              </section>
            </v-card-text>
          </template>
        </v-card>
      </v-container>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, mixins, Vue } from 'nuxt-property-decorator'
import ContestHeaderTab from '~/components/ContestHeaderTab.vue'
import ContestHeader from '~/components/ContestHeader.vue'
import { ProblemDetail } from '~/types/problem'
import MathJax from '~/mixins/mathjax'
import MixinContest from '~/mixins/contest'
import DifficultyChip from '~/components/parts/difficulty-chip.vue'
import Editor from '~/components/Editor.vue'
import languages from '~/assets/languages'
import { Language } from '~/types/language'

@Component({
  components: {
    DifficultyChip,
    ContestHeader,
    ContestHeaderTab,
    Editor
  },
  layout: 'contest'
})
export default class PageContestTasks extends mixins(MathJax, MixinContest) {
  problem: ProblemDetail | null = null
  language: Language | undefined = languages[0]
  submitted = false

  copy(text: string) {
    const tmp = document.createElement('textarea')
    tmp.style.position = 'fixed'
    tmp.style.right = '200%'
    tmp.textContent = text
    document.body.appendChild(tmp)
    tmp.select()
    document.execCommand('copy')
    document.body.removeChild(tmp)
  }

  created() {
    this.language =
      languages.filter(
        (lang) => localStorage.getItem('lang') === lang.innerName
      )[0] || languages[0]
    this.getContest()
    this.$api.Tasks.show(
      this.$route.params.contestName,
      this.$route.params.taskName
    ).then((ret: ProblemDetail) => {
      this.problem = ret
      this.$nextTick(() => {
        this.renderMathJax()
      })
    })
  }

  get source(): string {
    const editor = this.$refs.editor as Vue & {
      value: string
    }
    return editor.value
  }

  get selectableLanguages() {
    return languages.map((lang) => ({
      text: lang.name,
      value: lang
    }))
  }

  submit() {
    if (!/\S/.test(this.source)) {
      alert('空のソースは提出できません。')
      return
    }
    localStorage.setItem('lang', this.language!.innerName)
    this.submitted = true
    this.$api.Tasks.submit(
      this.contestSlug,
      this.$route.params.taskName,
      this.language!.innerName,
      this.source,
      'AC'
    )
      .then(() => {
        this.$router.push(`/contests/${this.contestSlug}/submits`)
      })
      .catch(() => {
        alert('提出に失敗しました。')
        this.submitted = false
      })
  }
}
</script>

<style scoped lang="scss">
.statement {
  margin: 1em 0 2em 1.5em;
  font-size: 1.15em;
}
.sample {
  &__head {
    display: flex;
    &__title {
      display: flex;
      align-items: center;
      font-size: 1.45rem;
      font-weight: bold;
      margin-right: 1rem;
    }
  }
  &__code {
    width: 100%;
    color: black;
    padding: 0.5em;
    margin: 0;
    font-size: 1em;
    font-weight: normal;
    &:before,
    &:after {
      content: none;
    }
  }

  .__editor {
    margin: 1em;
  }
}
.submit {
  &__head {
    display: flex;
    &__title {
      @extend .sample__head__title;
    }
    &__select {
      max-width: 20em;
    }
  }
}
h3 {
  font-size: 1.5rem;
}
</style>

<style lang="scss">
@import '~/styles/markdown.scss';
.statement {
  @include markdown();
}
</style>
