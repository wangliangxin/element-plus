<template>
  <button ref="_ref" :class="[
    ns.b(),
    ns.m(_type),
    ns.m(_size),
    ns.is('disabled', _disabled),
    ns.is('loading', loading),
    ns.is('plain', plain),
    ns.is('round', round),
    ns.is('circle', circle),
    ns.is('text', text),
    ns.is('link', link),
    ns.is('has-bg', bg),
  ]" :aria-disabled="_disabled || loading" :disabled="_disabled || loading" :autofocus="autofocus" :type="nativeType"
    :style="buttonStyle" @click="handleClick">
    <template v-if="loading">

      <!-- 当存在自定义loading组件，优先使用，否则使用默认loadingIcon  -->
      <slot v-if="$slots.loading" name="loading" />
      <el-icon v-else :class="ns.is('loading')">
        <component :is="loadingIcon" />
      </el-icon>
    </template>
    <el-icon v-else-if="icon || $slots.icon">
      <component :is="icon" v-if="icon" />
      <slot v-else name="icon" />
    </el-icon>
    <span v-if="$slots.default" :class="{ [ns.em('text', 'expand')]: shouldAddSpace }">
      <slot />
    </span>
  </button>
</template>

<script lang="ts" setup>
import { Text, computed, inject, ref, useSlots } from 'vue'
import { ElIcon } from '@element-plus/components/icon'
import {
  useDeprecated,
  useDisabled,
  useFormItem,
  useGlobalConfig,
  useNamespace,
  useSize,
} from '@element-plus/hooks'
import { buttonGroupContextKey } from '@element-plus/tokens'
import { buttonEmits, buttonProps } from './button'
import { useButtonCustomStyle } from './button-custom'

// 命名组件名，setup语法糖中 使用defineOptions定义
defineOptions({
  name: 'ElButton',
})

const props = defineProps(buttonProps)
const emit = defineEmits(buttonEmits)
const slots = useSlots()

// 弃用提醒
// 3.0.0 已经弃用了 type = text的按钮，使用link 替代， 任使用text将触发警告
useDeprecated(
  {
    from: 'type.text',
    replacement: 'link',
    version: '3.0.0',
    scope: 'props',
    ref: 'https://element-plus.org/en-US/component/button.html#button-attributes',
  },
  computed(() => props.type === 'text')
)

// 注入一个由祖先组件或整个应用 (通过 app.provide()) 提供的值。
// inject 有三个重载， inject("key"),  inject('key', 'defaultVal')
/**
 * 
 * <script setup>
import { inject } from 'vue'
import { fooSymbol } from './injectionSymbols'

// 注入值的默认方式
const foo = inject('foo')

// 注入响应式的值
const count = inject('count')

// 通过 Symbol 类型的 key 注入
const foo2 = inject(fooSymbol)

// 注入一个值，若为空则使用提供的默认值
const bar = inject('foo', 'default value')

// 注入一个值，若为空则使用提供的工厂函数
const baz = inject('foo', () => new Map())

// 注入时为了表明提供的默认值是个函数，需要传入第三个参数
const fn = inject('function', () => {}, false)
</script>
 */
const buttonGroupContext = inject(buttonGroupContextKey, undefined)

const globalConfig = useGlobalConfig('button')
const ns = useNamespace('button')
const { form } = useFormItem()
const _size = useSize(computed(() => buttonGroupContext?.size))
const _disabled = useDisabled()

const _ref = ref<HTMLButtonElement>()

const _type = computed(() => props.type || buttonGroupContext?.type || '')
const autoInsertSpace = computed(
  () => props.autoInsertSpace ?? globalConfig.value?.autoInsertSpace ?? false
)

// add space between two characters in Chinese
const shouldAddSpace = computed(() => {
  const defaultSlot = slots.default?.()
  if (autoInsertSpace.value && defaultSlot?.length === 1) {
    const slot = defaultSlot[0]
    if (slot?.type === Text) {
      const text = slot.children as string
      return /^\p{Unified_Ideograph}{2}$/u.test(text.trim())
    }
  }
  return false
})

const buttonStyle = useButtonCustomStyle(props)

const handleClick = (evt: MouseEvent) => {
  if (props.nativeType === 'reset') {
    form?.resetFields()
  }
  emit('click', evt)
}

defineExpose({
  /** @description button html element */
  ref: _ref,
  /** @description button size */
  size: _size,
  /** @description button type */
  type: _type,
  /** @description button disabled */
  disabled: _disabled,
  /** @description whether adding space */
  shouldAddSpace,
})
</script>
