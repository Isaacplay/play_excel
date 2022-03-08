<template>
  <div class="container">
    <div class="step-box" v-show="step == 1">
      <el-upload
        class="upload-demo"
        drag
        :before-upload='beforeUpload'
        action="https://jsonplaceholder.typicode.com/posts/"
        multiple>
        <i class="el-icon-upload"></i>
        <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
      </el-upload>
    </div>
    <div class="step-box" v-show="step == 2">
      <div>共有{{table_list.length}}条数据</div>
      <div>无名数据{{noname_list.length}}条</div>
      <div>无时间数据{{nodate_list.length}}条</div>
      <div>普通数据{{normal_list.length}}条</div>
      <div v-show="nodate_list.length > 0">存在无时间数据,请先输入补充的时间</div>
      <el-input v-show="nodate_list.length > 0" v-model="dataTime" placeholder="请输入修改的时间"></el-input>
      <el-button type="primary" plain @click="step3Action" >下一步</el-button>
    </div>
    <div class="step-box" v-show="step == 3">
      <div class="loaction-item" v-for="(item,index) in locationList" :key="index + 'location'">
        <div>地区: {{index}}</div>
        <div>共有: {{item.list.length}}条</div>
        <div>管控方式: 
          <el-radio-group v-model="item.type">
            <el-radio label="__EMPTY_8">集中隔离</el-radio>
            <el-radio label="__EMPTY_9">居家健康监测</el-radio>
            <el-radio label="__EMPTY_10">严格健康监测</el-radio>
            <el-radio label="__EMPTY_11">自我健康监测</el-radio>
            <el-radio label="__EMPTY_12">自由通行</el-radio>
          </el-radio-group>
        </div>
      </div>
      <el-button type="primary" plain @click="endAction" >生成</el-button>
    </div>
  </div>
</template>

<script type="text/ecmascript-6">
  import * as XLSX from 'xlsx';
  import excel from "./excel";
  export default {
    name: 'Index',
    data () {
      return {
        step:1,
        table_title:{},
        table_title_2:{},
        tableData:null,
        table_list:[],
        noname_list:[],
        nodate_list:[],
        normal_list:[],
        dataTime:'2.19',
        locationList:{},
      }
    },
    mounted () {
      
    },
    methods: {
      beforeUpload(file){
        console.log(file)
        this.readFile(file)
        return false
      },
      readFile(file) {
        const reader = new FileReader();
        reader.readAsArrayBuffer(file);
        reader.onloadstart = (e) => {
            this.tableLoading = true;
        };
        reader.onprogress = (e) => {
            this.progressPercent = Math.round((e.loaded / e.total) * 100);
        };
        reader.onerror = (e) => {
            this.$Message.error("文件读取出错");
        };
        reader.onload = (e) => {
            const data = e.target.result;
            const { header, results } = excel.read(data, "array");
            const tableTitle = header.map((item) => {
                return { title: item, key: item };
            });
            //这里的tableData就是拿到的excel表格中的数据
            // console.log(results)
            // console.log(tableTitle)
            this.tableData = results;
            this.step2Action();
            // this.tableTitle = tableTitle;
            // const map = new Map();
            // header.map((key) => {
            //     let arr = new Array()
            //     results.map((v) => {
            //         arr.push(v[key])
            //     });
            //     map.set(key,arr)
            // });
            // results.forEach(title=>{
            //
            // })
            // console.log(map)
            // console.log(map.get("ID"));
            // this.tableLoading = false;
        };
      },
      step2Action(){
        let list = this.tableData
        this.table_title = list.shift()
        this.table_title_2 = list.shift()
        this.table_list = list
        console.log(list)
        this.noname_list = [];  //无名数据
        this.nodate_list = [];  //无离开日期
        this.normal_list = [];  //普通日期
        for(let i in list){
          if(!list[i]['__EMPTY']){
            this.noname_list.push(list[i])
          }else if(!list[i]['__EMPTY_3']){
            this.nodate_list.push(list[i])
          }else{
            this.normal_list.push(list[i])
          }
        }
        this.step = 2;
      },
      step3Action(){
        let list = [];
        if(this.nodate_list.length > 0){
          for(let i in this.nodate_list){
            this.nodate_list[i]['__EMPTY_3'] = this.dataTime
          }
          list = this.normal_list.concat(this.nodate_list)
        }
        let locationMap = {};
        list.forEach((item)=>{
          let location = (item['__EMPTY_5'] || item['__EMPTY_6'] || item['__EMPTY_7']).replace('市','').replace('区','').replace('县','').replace('村','').replace('镇','')
          if(locationMap.hasOwnProperty(location)){
            console.log(locationMap[location])
            locationMap[location].list.push(item)
          }else{
            locationMap[location] = {};
            locationMap[location].list = [item]
            locationMap[location].type = '__EMPTY_9'
          }
        })
        this.locationList = locationMap
        this.step = 3;
      },
      endAction(){
        let list = [];
        let locationMap = this.locationList
        for(let i in locationMap){
          locationMap[i].list.forEach((item) =>{
            for(let k = 1; k < 23; k++){
                item['__EMPTY_'+k] = item['__EMPTY_'+k] || ' '
            }
            item['__EMPTY_8'] = ' '
            item['__EMPTY_9'] = ' '
            item['__EMPTY_10'] = ' '
            item['__EMPTY_11'] = ' '
            item['__EMPTY_12'] = ' '
            item[locationMap[i].type] = '√'
          })
          list = list.concat(locationMap[i].list)
        }
        // list.unshift(this.table_title_2)
        // list.unshift(this.table_title)
        console.log(list)
        const ws = XLSX.utils.json_to_sheet(list)
        // 新建book
        const wb = XLSX.utils.book_new()
        // 生成xlsx文件(book,sheet数据,sheet命名)
        XLSX.utils.book_append_sheet(wb, ws, '本街道在管')
        // 写文件(book,xlsx文件名称)
        XLSX.writeFile(wb, '导出.xlsx')
      }
    },
  }
</script>

<style scoped>
.container{
  width: 100%;
  height: 100%;
  padding: 10px 30px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.con-item{
  display: flex;
  justify-content: center;
  align-items: center;
  width: auto;
  height: 100px;
  padding: 20px;
  background-color: rgb(245,245,245);
  margin-right: 15px;
}
.loaction-item{
  margin: 24px 0;
}

</style>
