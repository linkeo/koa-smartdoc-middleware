<template>
  <div ref="container" class="search">
    <div class="input-wrap">
      <input
        ref="input"
        type="search"
        placeholder="搜索 API或文档"
        v-model="keyword"
      />
      <clear-button :show="!!keyword" @click="clear"></clear-button>
    </div>
    <div class="results-panel" :class="{ show: keyword }">
      <p class="empty" v-if="results.length === 0">无结果</p>
      <div
        v-for="(item, index) in results"
        :class="'matching-post ' + item.className"
        :key="index"
        @click="clear"
      >
        <router-link :to="item.link">
          <h2>
            <small v-if="item.type">{{ item.type }}</small>
            <span>{{ item.title }}</span>
          </h2>
          <p v-if="item.text">{{ item.text }}</p>
        </router-link>
      </div>
    </div>
  </div>
</template>

<script>
import Debug from 'debug';
import escapeRegexp from 'lodash.escaperegexp';
import ClearButton from './ClearButton.vue';

const debug = Debug('debug');

/** @param {string} keyword */
const makeSeqMatcher = keyword => {
  if (typeof keyword !== 'string') {
    return { seq: null, exec: null };
  }
  const seq = keyword.replace(/\s/g, '');
  if (seq.length === 0) {
    return { seq: null, exec: null };
  }
  const lowerChars = Array.from(seq.toLowerCase());
  const exec = input => {
    if (typeof input !== 'string' || input.length === 0) {
      return null;
    }
    const lowerInput = input.toLowerCase();
    const start = lowerInput.indexOf(lowerChars[0]);
    if (start === -1) {
      return null;
    }
    const spans = [lowerChars[0]];
    let lastIndex = start;
    let end = start + lowerChars[0].length;
    let conn = 0;
    for (const char of lowerChars.slice(1)) {
      const index = lowerInput.indexOf(char, lastIndex + 1);
      if (index === -1) {
        return null;
      }
      if (index === lastIndex + char.length) {
        spans[spans.length - 1] += char;
        conn += 1;
      } else {
        spans.push(char);
      }
      lastIndex = index;
      end = index + char.length;
    }
    if (conn + conn < seq.length) {
      return null;
    }
    return {
      0: input.slice(start, end),
      index: start,
      length: 1,
      input,
      spans,
    };
  };
  return { exec, seq };
};

const getOrder = result => {
  // 匹配到的完整字符串长度
  const a = result.input.length;
  // 匹配到的部分长度
  const b = result[0].length;
  // 匹配到的部分的连续子串个数
  const c = result.spans.length;
  // 加权计算分数
  const o = a * 2 + b * 7 + c * 97;
  // console.log(o, a, b, c, cl, result);
  return o;
};

export default {
  components: { ClearButton },
  props: ['spec'],
  data() {
    return {
      timer: null,
      keyword: '',
      results: [],
    };
  },
  mounted() {
    if (this.keyupListener) {
      window.removeEventListener('keyup', this.keyupListener);
    }
    this.keyupListener = event => {
      const inputable =
        document.designMode === 'on' ||
        event.target.isContentEditable ||
        /^(?:input|textarea|select)$/i.test(event.target.tagName);
      if (!inputable && event.key === 'f') {
        const { container, input } = this.$refs;
        if (
          container &&
          typeof container.scrollIntoViewIfNeeded === 'function'
        ) {
          container.scrollIntoViewIfNeeded();
        } else if (
          container &&
          typeof container.scrollIntoView === 'function'
        ) {
          container.scrollIntoView();
        }
        if (input) {
          input.focus();
          input.setSelectionRange(0, Number.MAX_SAFE_INTEGER);
        }
      }
    };
    window.addEventListener('keyup', this.keyupListener);
  },
  beforeDestroy() {
    if (this.keyupListener) {
      window.removeEventListener('keyup', this.keyupListener);
    }
  },
  methods: {
    clear() {
      this.keyword = '';
    },
    search(to) {
      const { seq, exec } = makeSeqMatcher(to);
      let maxLength = 90;
      const startTime = Date.now();
      const dueTime = startTime + 500;
      const timeout = () => Date.now() > dueTime;
      const delay = () => Date.now() - startTime;
      const execMatch = text => {
        const start = Date.now();
        const result = exec(text);
        const cost = Date.now() - start;
        if (cost >= 100) {
          console.log(
            `slow match total=${text.length} seq=${seq.length} cost=${cost}ms`,
          );
        }
        return result;
      };
      if (seq) {
        const results = [];
        for (const controller of (this.spec && this.spec.modules) || []) {
          if (timeout()) {
            break;
          }
          const matchController = execMatch(controller.title);
          if (matchController) {
            results.push({
              className: 'api controller',
              order: getOrder(matchController, seq),
              title: controller.title,
              text: '控制器',
              link: `/api/controller/${controller.name}`,
            });
          }
          for (const route of controller.actions || []) {
            if (timeout()) {
              break;
            }
            const matchAction =
              execMatch(route.title) ||
              execMatch(`${route.route.method} ${route.route.path}`);

            if (matchAction) {
              results.push({
                className: 'api action',
                order: getOrder(matchAction, seq),
                title: `${route.title}（${controller.title}）`,
                text: `接口：${route.route.method} ${route.route.path}`,
                link: `/api/controller/${controller.name}#${route.name}`,
              });
            }
          }
        }
        debug('api search cost ' + delay() + 'ms');

        /**
         * @param {string} title
         * @param {string[]} paragraphs
         */
        const match = (title, paragraphs) => {
          let matchContent = execMatch(title);
          for (const content of paragraphs) {
            if (!matchContent) {
              matchContent = execMatch(content);
            }
          }
          if (matchContent) {
            const content = matchContent.input;
            const index = Math.max(
              0,
              Math.min(
                matchContent.index - 20,
                content.length - maxLength + 20,
              ),
            );
            let prefix = index > 0 ? '…' : '';
            let suffix = index + maxLength < content.length ? '…' : '';
            return {
              order: getOrder(matchContent, seq),
              preview:
                prefix + content.slice(index, index + maxLength) + suffix,
            };
          }
          return null;
        };
        for (const group of (this.spec && this.spec.docs) || []) {
          for (const item of group.list) {
            if (timeout()) {
              break;
            }
            const itemMatch = match(item.title, item.content || []);
            if (itemMatch) {
              results.push({
                className: 'document toplevel',
                order: itemMatch.order,
                title: item.title,
                text: itemMatch.preview || item.title,
                link: item.link,
              });
            }
            for (const section of item.children) {
              if (timeout()) {
                break;
              }
              const sectionMatch = match(section.title, section.content || []);
              if (sectionMatch) {
                results.push({
                  className: 'document section',
                  order: sectionMatch.order,
                  title: item.title + ' • ' + section.title,
                  text: sectionMatch.preview || section.title,
                  link: section.link,
                });
              }
            }
          }
        }
        debug('api search + doc search cost ' + delay() + 'ms');
        results.sort((a, b) => a.order - b.order);
        debug(
          results.length + ' results cost ' + (Date.now() - startTime) + 'ms',
        );
        this.results = results.slice(0, 100);
        debug('top 100 orders ' + this.results.map(x => x.order));
      } else {
        this.results = [];
      }
    },
  },
  watch: {
    keyword(to) {
      // -- check input interval
      // const now = Date.now();
      // if (this.timer) {
      //   console.log(now - this.timer);
      // }
      // this.timer = now;

      // -- debounced search
      if (this.timer) {
        clearTimeout(this.timer);
        this.timer = null;
      }
      this.timer = setTimeout(() => {
        this.search(to);
        this.timer = null;
      }, 350);
    },
  },
};
</script>
