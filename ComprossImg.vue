<template>
  <div class="">
    <el-upload
      class="compress-upload"
      drag
      action="https://jsonplaceholder.typicode.com/posts/"
      :auto-upload="false"
      :show-file-list="false"
      accept=".png,.jpg,.jpeg"
      :on-change="handleChange"
      multiple>
      <i class="el-icon-upload"></i>
      <div class="el-upload__text">将要压缩的图片拖到此处，或<em>点击上传</em>要压缩的图片</div>
      <div class="el-upload__tip">只能压缩jpg/png/jpeg文件</div>
      <div class="el-upload__tip">请先设置参数再上传图片，上传之后自动压缩</div>
    </el-upload>
    
    <div class="compress-persents-wrap">
      <span>压缩率：</span>
      <el-input class="compress-persents" v-model="compressPersents"></el-input>
      <span>图片类型：</span>
      <el-select v-model="imgType" class="compress-persents">
        <el-option
          v-for="item in options"
          :key="item.value"
          :label="item.value"
          :value="item.value">
        </el-option>
      </el-select>
      <p style="margin-top: 10px"></p>
      <span>宽度：</span>
      <el-input class="compress-persents" v-model="imgWidth"></el-input>
      <span>高度：</span>
      <el-input class="compress-persents" v-model="imgHeight"></el-input>
      <p>可以从 0 到 1 的区间内选择图片的质量。如果超出取值范围，将会使用默认值 0.5。</p>
      <p>可以统一设置图片宽高，必须大于0，小于等于0默认为图片宽高。只设置一个会按照比例设置宽高。</p>
      <p>注意，压缩率是根据图片质量进行压缩的。图片类型origin表示原始图片类型</p>
    </div>
    
    <ul class="compress-wrap" :style="{height: height}">
      <li v-for="(val, index) in fileList" :key="val.name + index" class="upload-item">
        <p class="name">{{val.name}}</p>
        <p class="size">{{val.originalSize}}</p>
        <p class="status" :class="'status' + val.status">{{val.status == 1 ? 'compress' : 'success'}}</p>
        <p class="size">{{val.compressSize}}</p>
        <p class="persents"><span v-if="val.persents">-{{val.persents}}%</span></p>
        <p v-if="val.status == 2" @click="download(val)" class="download">download</p>
      </li>
    </ul>
    <el-row style="text-align: center"><el-button @click="batchDownload">download all</el-button></el-row>
    
  </div>
</template>
<script>
  import JSZip from 'jszip'
  import 'file-saver'
  export default {
    name: '',
    components: {},
    data() {
      return {
        fileList: [],
        compressPersents: 0.5,
        imgWidth: 0,
        imgHeight: 0,
        imgType: 'origin',
        options: [{
          value: 'origin'
        },{
          value: 'png'
        },{
          value: 'jpg'
        },{
          value: 'jpeg'
        }],
        selectImgEnd: false,
        currentIndex: 0
      }
    },
    methods: {
      handleChange(res) {
        if(res.size / 1024 / 1024 > 1){
          res.originalSize = (res.size / 1024 / 1024).toFixed(2) + 'M';
        }else{
          res.originalSize = (res.size / 1024).toFixed(2) + 'kb';
        }
        res.status = 1;
        res.name = this.imgType == 'origin' ? res.name : res.name.split('.')[0] + '.' + this.imgType;
        this.fileList.push(res);
        if (!this.selectImgEnd) {
          this.selectImgEnd = true;
          setTimeout(() => {
            this.startCompress();
            this.selectImgEnd = false;
          }, 300)
        }
      },
      //开始压缩
      startCompress(){
        for (let i = this.currentIndex; i < this.fileList.length; i++) {
          let item = this.fileList[i];
          this.compressImg(item).then(res => {
            this.fileList[i].status = 2;
            if(res.size / 1024 / 1024 > 1){
              this.fileList[i].compressSize = (res.size / 1024 / 1024).toFixed(2) + 'M';
            }else{
              this.fileList[i].compressSize = (res.size / 1024).toFixed(2) + 'kb';
            }
            this.fileList[i].persents = ((1 - res.size / this.fileList[i].size) * 100).toFixed(2);
            this.fileList[i].blob = res;
          })
        }
        this.currentIndex = this.fileList.length;
      },
      //压缩图片
      compressImg(res) {
        return new Promise((resolve, reject) => {
          let imgUrl = URL.createObjectURL(res.raw);
          var img = new Image();
          img.src = imgUrl;
          img.onload = () => {
            let canvas = this.imagetoCanvas(img);
            if(this.compressPersents <= 0 && this.compressPersents >= 1){
              this.compressPersents = 0.5;
            };
            let blob = this.base64ToBlob(canvas.toDataURL('image/jpeg', this.compressPersents));
            resolve(blob);
            // canvas.toBlob(blob => {
            //   resolve(blob);
            // }, 'image/jpeg', this.compressPersents);
            canvas.remove();
          }
        })
      },
      //base64转化成blob
      base64ToBlob(code) {
        let parts = code.split(';base64,');
        let contentType = parts[0].split(':')[1];
        let raw = window.atob(parts[1]);
        let rawLength = raw.length;
        let uInt8Array = new Uint8Array(rawLength);
        for (let i = 0; i < rawLength; ++i) {
          uInt8Array[i] = raw.charCodeAt(i);
        }
        return new Blob([uInt8Array], {type: contentType});
      },
      //创建canvas
      imagetoCanvas(image) {
        var cvs = document.createElement("canvas");
        var ctx = cvs.getContext('2d');
        let width = image.width;
        let height = image.height;
        let aspectRatio = image.width / image.height;
        if(this.imgWidth > 0 && this.imgHeight > 0){
          width = this.imgWidth;
          height = this.imgHeight;
        }else if(this.imgWidth > 0){
          width = this.imgWidth;
          height = this.imgWidth / aspectRatio;
        }else if(this.imgHeight > 0){
          width = this.imgHeight * aspectRatio;
          height = this.imgHeight;
        }
        cvs.width = width;
        cvs.height = height;
        ctx.drawImage(image, 0, 0, cvs.width, cvs.height);
        return cvs ;
      },
      //单张下载
      download({name, blob}) {
        const link = document.createElement("a");
        link.href = URL.createObjectURL(blob);
        link.download = name;
        link.click();
        link.remove();
        URL.revokeObjectURL(link.href);
      },
      //批量下载
      batchDownload(){
        let zip = new JSZip();
        var img = zip.folder("images");
        for (let i = 0; i < this.fileList.length; i++) {
          let item = this.fileList[i];
          img.file(item.name, item.blob);
        };
        zip.generateAsync({type:"blob"})
          .then(function(content) {
            saveAs(content, "images.zip");
          });
      }
    },
    computed: {
      height() {
        return window.innerHeight - 560 + 'px'
      },
    },
  }
</script>
<style lang="scss" type="text/scss" scoped>
  .compress-upload {
    width: 600px;
    height: 200px;
    margin: 20px auto;
  }
  .compress-persents-wrap{
    width: 700px;
    margin: 0 auto;
    span{
      display: inline-block;
      width: 70px;
      text-align: right;
    }
  }
  .compress-persents{
    width: 200px;
    margin-right: 10px;
  }
  .compress-wrap{
    overflow-y: auto;
    width: 800px;
    margin: 10px auto;
  }
  .upload-item{
    display: flex;
    padding: 2px 10px;
    margin-top: 5px;
    border-radius: 3px;
    background: #F0F0F0;
    line-height: 24px;
    .name{
      width: 210px;
      @include line-clamp(1);
    }
    .status{
      width: 200px;
      height: 20px;
      margin: 2px 0;
      border-radius: 20px;
      text-align: center;
      color: #fff;
      line-height: 20px;
      font-size: 14px;
    }
    .size{
      width: 60px;
      margin: 0 10px;
      text-align: center;
    }
    .persents{
      width: 80px;
      margin: 0 20px 0 35px;
      text-align: right;
    }
    .download{
      font-size: 16px;
      font-weight: bold;
      cursor: pointer;
    }
    .status1{
      background: #0AA574;
    }
    .status2{background: #8CE413}
  }
</style>