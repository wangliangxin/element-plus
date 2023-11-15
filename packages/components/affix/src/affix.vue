<!-- template 解析 start -->
<!-- 1. ns 是通过 useNamespace 返回的一个实例对象 -->
<!--    未手动定义配置全局选项的namespace时，默认为 'el' -->
<!--    ns.b() = el-affix -->
<!--    ns.m('fixed') = -->
<!-- template 解析 end -->
<template>
  <div ref="root" :class="ns.b()" :style="rootStyle">
    <div :class="{ [ns.m('fixed')]: fixed }" :style="affixStyle">
      <slot />
    </div>
  </div>
</template>

<script lang="ts" setup>
import { computed, onMounted, ref, shallowRef, watch, watchEffect } from 'vue'
import {
  useElementBounding,
  useEventListener,
  useWindowSize,
} from '@vueuse/core'
import { addUnit, getScrollContainer, throwError } from '@element-plus/utils'
import { useNamespace } from '@element-plus/hooks'
import { affixEmits, affixProps } from './affix'
import type { CSSProperties } from 'vue'

const COMPONENT_NAME = 'ElAffix'
defineOptions({
  name: COMPONENT_NAME,
})

const props = defineProps(affixProps)
const emit = defineEmits(affixEmits)

const ns = useNamespace('affix') // block 名为 affix

// 使用shallowRef 而不是 ref，减少深层监听，提高性能
const target = shallowRef<HTMLElement>()
const root = shallowRef<HTMLDivElement>()
const scrollContainer = shallowRef<HTMLElement | Window>() // 滚动容器
const { height: windowHeight } = useWindowSize() // 得到视口高度 （）

// root 未挂载的时候 下面这些都是 0，当root 挂载后，会得到对应的值
const {
  height: rootHeight,
  width: rootWidth,
  top: rootTop,
  bottom: rootBottom,
  update: updateRoot,
} = useElementBounding(root, { windowScroll: false })
const targetRect = useElementBounding(target)

const fixed = ref(false)
const scrollTop = ref(0)
const transform = ref(0)

// 设置root的样式
const rootStyle = computed<CSSProperties>(() => {
  return {
    height: fixed.value ? `${rootHeight.value}px` : '',
    width: fixed.value ? `${rootWidth.value}px` : '',
  }
})

// 设置固钉的样式
// 注意，这里虽然设置了 top 和bottom这些，但是只有在 fixed 的时候才会生效
// el-affix  ->  position: fixed;
const affixStyle = computed<CSSProperties>(() => {
  if (!fixed.value) return {}

  const offset = props.offset ? addUnit(props.offset) : 0
  return {
    height: `${rootHeight.value}px`,
    width: `${rootWidth.value}px`,
    top: props.position === 'top' ? offset : '',
    bottom: props.position === 'bottom' ? offset : '',
    // 这里transform用于指定target的时候生效
    transform: transform.value ? `translateY(${transform.value}px)` : '',
    // 层级设置，这里的默认值为200
    zIndex: props.zIndex,
  }
})

// 更新状态，主要用于更新 fixed.value
const update = () => {
  if (!scrollContainer.value) return

  // instanceof是一个用于检查对象是否属于特定类型的运算符。它的语法是object instanceof constructor，
  // 其中object是要检查的对象，constructor是要检查的类型。
  scrollTop.value =
    scrollContainer.value instanceof Window
      ? document.documentElement.scrollTop
      : scrollContainer.value.scrollTop || 0

  // 吸顶
  if (props.position === 'top') {
    if (props.target) {
      const difference =
        targetRect.bottom.value - props.offset - rootHeight.value
      fixed.value = props.offset > rootTop.value && targetRect.bottom.value > 0
      transform.value = difference < 0 ? difference : 0
    } else {
      // 吸底
      // 当rootTop（距离顶部的距离）小于offset（设定的偏移距离）时固定
      fixed.value = props.offset > rootTop.value
    }
  } else if (props.target) {
    const difference =
      windowHeight.value -
      targetRect.top.value -
      props.offset -
      rootHeight.value
    fixed.value =
      windowHeight.value - props.offset < rootBottom.value &&
      windowHeight.value > targetRect.top.value
    transform.value = difference < 0 ? -difference : 0
  } else {
    fixed.value = windowHeight.value - props.offset < rootBottom.value
  }
}

const handleScroll = () => {
  updateRoot()
  emit('scroll', {
    scrollTop: scrollTop.value,
    fixed: fixed.value,
  })
}

watch(fixed, (val) => emit('change', val))

onMounted(() => {
  if (props.target) {
    target.value =
      document.querySelector<HTMLElement>(props.target) ?? undefined
    if (!target.value)
      throwError(COMPONENT_NAME, `Target is not existed: ${props.target}`)
  } else {
    target.value = document.documentElement
  }
  scrollContainer.value = getScrollContainer(root.value!, true)
  updateRoot()
})

useEventListener(scrollContainer, 'scroll', handleScroll)
watchEffect(update)

defineExpose({
  /** @description update affix status */
  update,
  /** @description update rootRect info */
  updateRoot,
})
</script>
