<script>
export default {
  props: {
    value: {
      type:     String,
      default: ''
    },
    row: {
      type:     Object,
      required: true
    },
  },
  data() {
    return { url: '' };
  },
  mounted() {
    this.findUrl();
  },
  methods: {
    findUrl() {
      this.url = `/deviceProtocol/${ this.value }`;
    }
  }
};
</script>

<template>
  <span>
    <nuxt-link :to="url">
      {{ value }}
    </nuxt-link>
  </span>
</template>
