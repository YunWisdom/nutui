<template>
  <view
    ref="container"
    :class="classes"
    @touchstart="onTouchStart"
    @touchmove="onTouchMove"
    @touchend="onTouchEnd"
    @touchcancel="onTouchEnd"
  >
    <view
      :class="{
        [`${componentName}-inner`]: true,
        [`${componentName}-vertical`]: isVertical
      }"
      :style="state.style"
    >
      <slot></slot>
    </view>
    <slot name="page"></slot>
    <view
      :class="{
        [`${componentName}-pagination`]: true,
        [`${componentName}-pagination-vertical`]: isVertical
      }"
      v-if="paginationVisible && !slots.page"
    >
      <i
        :style="{
          backgroundColor: activePagination === index ? paginationColor : '#ddd'
        }"
        v-for="(item, index) in state.children.length"
        :key="index"
      />
    </view>
  </view>
</template>

<script lang="ts">
import {
  onMounted,
  onActivated,
  onDeactivated,
  onBeforeUnmount,
  provide,
  ComponentInternalInstance,
  ComponentPublicInstance,
  reactive,
  computed,
  nextTick,
  ref,
  watch
} from 'vue';
import { createComponent } from '@/utils/create';
import { useTouch } from './use-touch';
import { useExpose } from './use-expose';
const { create, componentName } = createComponent('swiper');
import swiperItem from '@/packages/swiperitem/index.vue';
export default create({
  children: [swiperItem],
  props: {
    width: {
      type: [Number, String],
      default: window.innerWidth
    },
    height: {
      type: [Number, String],
      default: 0
    },
    direction: {
      type: [String],
      default: 'horizontal' //horizontal and vertical
    },
    paginationVisible: {
      type: Boolean,
      default: false
    },
    paginationColor: {
      type: String,
      default: '#fff'
    },
    loop: {
      type: Boolean,
      default: true
    },
    duration: {
      type: [Number, String],
      default: 500
    },
    autoPlay: {
      type: [Number, String],
      default: 0
    },
    initPage: {
      type: [Number, String],
      default: 0
    },
    touchable: {
      type: Boolean,
      default: true
    },
    isPreventDefault: {
      type: Boolean,
      default: true
    },
    isStopPropagation: {
      type: Boolean,
      default: true
    }
  },
  emits: ['change'],

  setup(props, { emit, slots }) {
    const container = ref();
    const state = reactive({
      active: 0,
      num: 0,
      rect: null as DOMRect | null,
      width: 0,
      height: 0,
      moving: false,
      offset: 0,
      touchTime: 0,
      autoplayTimer: 0 as number | undefined,
      children: [] as ComponentPublicInstance[],
      style: {}
    });

    const touch = useTouch();

    const classes = computed(() => {
      const prefixCls = componentName;
      return {
        [prefixCls]: true
      };
    });

    const isVertical = computed(() => props.direction === 'vertical');

    const delTa = computed(() => {
      return isVertical.value ? touch.state.deltaY : touch.state.deltaX;
    });

    const isCorrectDirection = computed(() => {
      return touch.state.direction === props.direction;
    });

    const childCount = computed(() => state.children.length);

    const size = computed(() => state[isVertical.value ? 'height' : 'width']);

    const trackSize = computed(() => childCount.value * size.value);

    const minOffset = computed(() => {
      if (state.rect) {
        const base = isVertical.value ? state.rect.height : state.rect.width;
        return base - size.value * childCount.value;
      }
      return 0;
    });

    const activePagination = computed(
      () => (state.active + childCount.value) % childCount.value
    );

    const getStyle = () => {
      state.style = {
        transitionDuration: `${state.moving ? 0 : props.duration}ms`,
        transform: `translate${isVertical.value ? 'Y' : 'X'}(${
          state.offset
        }px)`,
        [isVertical.value ? 'height' : 'width']: `${size.value *
          childCount.value}px`,
        [isVertical.value ? 'width' : 'height']: `${
          isVertical.value ? state.width : state.height
        }px`
      };
    };

    const relation = (child: ComponentInternalInstance) => {
      if (child.proxy) {
        state.children.push(child.proxy);
      }
    };

    const range = (num: number, min: number, max: number) => {
      return Math.min(Math.max(num, min), max);
    };

    const requestFrame = (fn: FrameRequestCallback) => {
      window.requestAnimationFrame.call(window, fn);
    };

    const getOffset = (active: number, offset = 0) => {
      let currentPosition = active * size.value;
      if (!props.loop) {
        currentPosition = Math.min(currentPosition, -minOffset.value);
      }

      let targetOffset = offset - currentPosition;
      if (!props.loop) {
        targetOffset = range(targetOffset, minOffset.value, 0);
      }

      return targetOffset;
    };

    const getActive = (pace: number) => {
      const { active } = state;
      if (pace) {
        if (props.loop) {
          return range(active + pace, -1, childCount.value);
        }
        return range(active + pace, 0, childCount.value - 1);
      }
      return active;
    };

    const move = ({ pace = 0, offset = 0, isEmit = false }) => {
      if (childCount.value <= 1) return;

      const { active } = state;

      const targetActive = getActive(pace);
      const targetOffset = getOffset(targetActive, offset);

      if (props.loop) {
        if (state.children[0] && targetOffset !== minOffset.value) {
          const rightBound = targetOffset < minOffset.value;
          (state.children[0] as any).setOffset(
            rightBound ? trackSize.value : 0
          );
        }
        if (state.children[childCount.value - 1] && targetOffset !== 0) {
          const leftBound = targetOffset > 0;
          (state.children[childCount.value - 1] as any).setOffset(
            leftBound ? -trackSize.value : 0
          );
        }
      }

      state.active = targetActive;
      state.offset = targetOffset;

      if (isEmit && active !== state.active) {
        emit('change', activePagination.value);
      }

      getStyle();
    };

    const resettPosition = () => {
      state.moving = true;

      if (state.active <= -1) {
        move({ pace: childCount.value });
      }
      if (state.active >= childCount.value) {
        move({ pace: -childCount.value });
      }
    };

    const stopAutoPlay = () => {
      clearTimeout(state.autoplayTimer);
    };

    const prev = () => {
      resettPosition();
      touch.reset();

      requestFrame(() => {
        requestFrame(() => {
          state.moving = false;
          move({
            pace: -1,
            isEmit: true
          });
        });
      });
    };

    const next = () => {
      resettPosition();
      touch.reset();

      requestFrame(() => {
        requestFrame(() => {
          state.moving = false;
          move({
            pace: 1,
            isEmit: true
          });
        });
      });
    };

    const to = (index: number) => {
      resettPosition();

      touch.reset();

      requestFrame(() => {
        requestFrame(() => {
          state.moving = false;
          let targetIndex;
          if (props.loop && childCount.value === index) {
            targetIndex = state.active === 0 ? 0 : index;
          } else {
            targetIndex = index % childCount.value;
          }
          move({
            pace: targetIndex - state.active,
            isEmit: true
          });
        });
      });
    };

    const autoplay = () => {
      if (props.autoPlay <= 0 || childCount.value <= 1) return;
      stopAutoPlay();

      state.autoplayTimer = setTimeout(() => {
        next();
        autoplay();
      }, Number(props.autoPlay));
    };

    const init = (active: number = +props.initPage) => {
      stopAutoPlay();
      state.rect = container.value.getBoundingClientRect();
      active = Math.min(childCount.value - 1, active);
      state.width = props.width ? +props.width : (state.rect as DOMRect).width;
      state.height = props.height
        ? +props.height
        : (state.rect as DOMRect).height;
      state.active = active;
      state.offset = getOffset(state.active);
      state.moving = true;
      getStyle();

      autoplay();
    };

    const onTouchStart = (e: TouchEvent) => {
      if (props.isPreventDefault) e.preventDefault();
      if (props.isStopPropagation) e.stopPropagation();
      if (!props.touchable) return;
      touch.start(e);
      state.touchTime = Date.now();
      stopAutoPlay();
      resettPosition();
    };

    const onTouchMove = (e: TouchEvent) => {
      if (props.touchable && state.moving) {
        touch.move(e);
        if (isCorrectDirection.value) {
          move({
            offset: delTa.value
          });
        }
      }
    };

    const onTouchEnd = (e: TouchEvent) => {
      if (!props.touchable || !state.moving) return;
      const speed = delTa.value / (Date.now() - state.touchTime);
      const isShouldMove =
        Math.abs(speed) > 0.3 ||
        Math.abs(delTa.value) > +(size.value / 2).toFixed(2);

      if (isShouldMove && isCorrectDirection.value) {
        let pace = 0;
        const offset = isVertical.value
          ? touch.state.offsetY
          : touch.state.offsetX;
        if (props.loop) {
          pace = offset > 0 ? (delTa.value > 0 ? -1 : 1) : 0;
        } else {
          pace = -Math[delTa.value > 0 ? 'ceil' : 'floor'](
            delTa.value / size.value
          );
        }
        move({
          pace,
          isEmit: true
        });
      } else if (delTa.value) {
        move({ pace: 0 });
      }
      state.moving = false;
      getStyle();
      autoplay();
    };

    provide('parent', {
      props,
      size,
      relation
    });

    useExpose({
      prev,
      next,
      to
    });

    onMounted(() => {
      nextTick(() => {
        init();
      });
    });

    onActivated(() => {
      nextTick(() => {
        init();
      });
    });

    onDeactivated(() => {
      stopAutoPlay();
    });

    onBeforeUnmount(() => {
      stopAutoPlay();
    });

    watch(
      () => props.initPage,
      val => {
        nextTick(() => {
          init(Number(val));
        });
      }
    );

    watch(
      () => state.children.length,
      () => {
        nextTick(() => {
          init(state.active);
        });
      }
    );

    watch(
      () => props.autoPlay,
      val => {
        val > 0 ? autoplay() : stopAutoPlay();
      }
    );

    return {
      state,
      classes,
      container,
      componentName,
      isVertical,
      slots,
      activePagination,
      onTouchStart,
      onTouchMove,
      onTouchEnd
    };
  }
});
</script>

<style scoped lang="scss">
@import 'index.scss';
</style>
