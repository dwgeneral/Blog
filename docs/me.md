<script>
function copyText() {
  var copyTextarea = document.getElementById("wechat");
  copyTextarea.select();
  document.execCommand("copy");
  alert("文本已复制到剪贴板！");
}
var image = document.getElementById("wechat");
image.addEventListener("click", func(){
    copyText()
});
</script>

<style>
img {
  pointer-events: none;
}
</style>


<div style="background-color: #fbfaf8;">

<center><img width="180px" style="border-radius: 50%; margin-top: 30px;" bor src="assets/me/avatar-1.jpeg"></center>

<h2 style="text-align: center;">👨‍💻Happy Engineer</h2>

<p style="text-align: center;"> 🤾‍♂️ Hi, I'm Weidong, base 北京 </p>

<p style="text-align: center;"> 🌟 9 年后端开发经验，擅长 golang/ruby 微服务开发 </p>

<p style="text-align: center;"> 🏂 平时也会玩一些 Node/Swift 全栈开发, 懂一些产品设计和数据分析 </p>

<p style="text-align: center;"> 🚴‍♂️ 喜欢探究业务背后的需求本质，然后用合适的技术手段满足它 </p>

<p style="text-align: center;"> 🧗‍♂️ 工作中常用的技术栈有 Redis, MySQL, MongoDB, Kafka, Docker, K8S, AWS Infras, Terraform </p>

<h2 style="text-align: center;">感兴趣的业务领域</h2>
<center>
    <p style="text-align: center;"> AIGC 应用落地探索 </p>
    <p style="text-align: center;"> SaaS 出海服务探索 </p>
    <p style="text-align: center;"> 医疗与少儿教育板块 </p>
</center>

<h2 style="text-align: center;">有经验的业务领域</h2>
<center>
    <p style="text-align: center;"> 电商亿级商品菜单数据系统设计 </p>
    <p style="text-align: center;"> 知识付费社群产品从0-100 </p>
    <p style="text-align: center;"> 微服务 DevOps 系统调优 </p>
</center>

<div style="display: flex; justify-content: center;">
    <img id="wechat" width="30px" style="margin: 10%;" bor src="assets/me/wechat.png">
    <img id="email" width="30px" style="margin: 10%;" bor src="assets/me/email.png">
    <img id="redbook" width="30px" style="margin: 10%;" bor src="assets/me/red.png">
</div>

</div>