<template>
  <view :class="classes" :style="{ height: pxCheck(buttonSize) }">
    <nut-icon
      name="minus"
      class="nut-inputnumber__icon"
      :class="{ 'nut-inputnumber__icon--disabled': !reduceAllow() }"
      :size="buttonSize"
      @click="reduce"
    >
    </nut-icon>
    <input
      type="number"
      :min="min"
      :max="max"
      :style="{ width: pxCheck(inputWidth) }"
      :disabled="disabled"
      :readonly="readonly"
      :value="modelValue"
      @input="change"
      @blur="blur"
      @focus="focus"
    />
    <nut-icon
      name="plus"
      class="nut-inputnumber__icon"
      :class="{ 'nut-inputnumber__icon--disabled': !addAllow() }"
      :size="buttonSize"
      @click="add"
    >
    </nut-icon>
  </view>
</template>
<script lang="ts">
import { computed } from 'vue';
import { createComponent } from '@/utils/create';
import { pxCheck } from '@/utils/pxCheck';
const { componentName, create } = createComponent('inputnumber');
export default create({
  props: {
    modelValue: {
      type: [Number, String],
      default: 0
    },
    inputWidth: {
      type: [Number, String],
      default: ''
    },
    buttonSize: {
      type: [Number, String],
      default: ''
    },
    min: {
      type: [Number, String],
      default: 1
    },
    max: {
      type: [Number, String],
      default: 9999
    },
    step: {
      type: [Number, String],
      default: 1
    },
    decimalPlaces: {
      type: [Number, String],
      default: 0
    },
    disabled: {
      type: Boolean,
      default: false
    },
    readonly: {
      type: Boolean,
      default: false
    }
  },
  emits: [
    'update:modelValue',
    'change',
    'blur',
    'focus',
    'reduce',
    'add',
    'overlimit'
  ],

  setup(props, { emit }) {
    const classes = computed(() => {
      const prefixCls = componentName;
      return {
        [prefixCls]: true,
        [`${prefixCls}--disabled`]: props.disabled
      };
    });

    const fixedDecimalPlaces = (v: string | number): string => {
      return Number(v).toFixed(Number(props.decimalPlaces));
    };

    const change = (event: Event) => {
      const input = event.target as HTMLInputElement;
      emit('update:modelValue', input.valueAsNumber, event);
    };

    const emitChange = (value: string | number, event: Event) => {
      let output_value: number | string = fixedDecimalPlaces(value);
      emit('change', output_value, event);
      emit('update:modelValue', output_value, event);
    };

    const addAllow = (value = Number(props.modelValue)): boolean => {
      return value < Number(props.max) && !props.disabled;
    };

    const reduceAllow = (value = Number(props.modelValue)): boolean => {
      return value > Number(props.min) && !props.disabled;
    };

    const reduce = (event: Event) => {
      emit('reduce', event);
      if (reduceAllow()) {
        let output_value = Number(props.modelValue) - Number(props.step);
        emitChange(output_value, event);
      } else {
        emit('overlimit', event);
      }
    };

    const add = (event: Event) => {
      emit('add', event);
      if (addAllow()) {
        let output_value = Number(props.modelValue) + Number(props.step);
        emitChange(output_value, event);
      } else {
        emit('overlimit', event);
      }
    };

    const focus = (event: Event) => {
      if (props.disabled) return;
      if (props.readonly) return;
      emit('focus', event);
    };

    const blur = (event: Event) => {
      if (props.disabled) return;
      if (props.readonly) return;
      const input = event.target as HTMLInputElement;

      let value = input.valueAsNumber;

      if (value < Number(props.min)) {
        value = Number(props.min);
      } else if (value > Number(props.max)) {
        value = Number(props.max);
      }
      emitChange(value, event);
      emit('blur', event);
    };

    return {
      classes,
      change,
      blur,
      focus,
      add,
      addAllow,
      reduce,
      reduceAllow,
      pxCheck
    };
  }
});
</script>

<style lang="scss">
@import 'index.scss';
</style>
