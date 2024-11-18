<script setup lang="ts">
import { onShow, onHide } from "@dcloudio/uni-app";
onShow(() => {
  console.log("App Show");
});
onHide(() => {
  console.log("App Hide");
});
</script>

<style>
page {
  background: #000 !important;
}
</style>
