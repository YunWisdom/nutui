<template>
  <div class="demo">
    <h2>基础用法</h2>
    <nut-cell>
      <nut-button type="primary" @click="state.show = true"
        >显示遮罩层</nut-button
      >
      <nut-overlay v-model:visible="state.show" :z-index="2000"></nut-overlay>
    </nut-cell>
    <h2>嵌套内容</h2>
    <nut-cell>
      <nut-button type="success" @click="state.show2 = true"
        >嵌套内容</nut-button
      >
      <nut-overlay v-model:visible="state.show2" :z-index="2000">
        <div class="wrapper">
          <div class="content">这里是正文</div>
        </div>
      </nut-overlay>
    </nut-cell>
  </div>
</template>

<script lang="ts">
import { reactive } from 'vue';
import { createComponent } from '@/utils/create';
const { createDemo } = createComponent('overlay');
export default createDemo({
  props: {},
  setup() {
    const state = reactive({
      show: false,
      show2: false
    });
    return {
      state
    };
  }
});
</script>

<style lang="scss" scoped>
.wrapper {
  display: flex;
  height: 100%;
  align-items: center;
  justify-content: center;
  .content {
    display: flex;
    width: 150px;
    height: 150px;
    background: #fff;
    border-radius: 8px;
    align-items: center;
    justify-content: center;
    color: red;
  }
}
</style>
