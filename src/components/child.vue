<!--
此处填写文件描述
@Author: linqian
@Date: 2020/6/9
-->
<template>
  <div>
      子组件
      <slot v-bind:user="user"></slot>
      </div>
</template>

<script>
    //import x from ''
    export default {
        props: {
            msg: String
        },
        components: {

        },
        data () {
            return {

            };
        },
        computed: {

        },
        methods: {

        },
        created() {

        },
        mounted() {

        }
}
</script>

<style lang='scss'>
</style>