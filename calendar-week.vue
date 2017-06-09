<template>
  <div class="calendar">
    <div class="operate">
      <div class="project">
        <i-select :model.sync="currentProject" v-ref:project @on-change="changeProject">
          <i-option v-for="(index, item) in projectList" :value="item.value">{{item.label}}</i-option>
        </i-select>
      </div>
      <div class="last-next-week">
        <span @click="lastWeek"><Icon type="arrow-left-b" size="30" color="#037ce4"></Icon></span>
        <span class="current-week">第 {{currentWeekNumber}} 周</span>
        <span @click="nextWeek"><Icon type="arrow-right-b" size="30" color="#037ce4"></Icon></span>
      </div>
      <a class="toggle calendar-icon-day" v-link="{name: 'calendarDay', query: {'userId': userId}}"></a>
    </div>
    <div class="weeks">
      <ul class="weeks-content" 
          @touchstart="horizontalTouchStart($event)" 
          @touchmove="horizontalTouchMove($event)" 
          @touchend="portraitTouchEnd"  
          :style="{left: weeksContentFactLeft+'px'}">
        <li></li>
        <li v-for="(index,item) in weeks">星期{{item}}</li>
      </ul>
    </div>
    <div class="calendar-content">
      <div class="calendar-time">
        <p v-for="(index,item) in 24">{{index}}:00</p>
      </div>
      <div class="calendar-matter">
        <ul class="calendar-matter-content" 
            @touchstart="horizontalTouchStart($event)" 
            @touchmove="horizontalTouchMove($event)" 
            @touchend="portraitTouchEnd" 
            :class="isPortraitScroll ? 'calendar-matter-moving' : ''" 
            :style="{left: weeksContentFactLeft + 'px'}">
          <li v-for="(index, item) in partnerRows" class="partner-content">
            <div class="partner-cont" 
                 v-for="(index2, item2) in item.hours" 
                 @click="partnerDetail(index, index2)">
              <div class="partner-con">
                <span v-for="(index3, item3) in item2.calendars">{{index3 + 1}}.{{item3.nickName}}</span>
                <span v-if="item2.thanTwo">
                  <Icon type="navicon" color="#999"></Icon>
                </span>
              </div>
              <div class="partner-popup" v-show="item2.popup">
                <p v-for="(index3, item3) in popupPartnerDetail.calendars">{{index3 + 1}}.{{item3.nickName}}</p>
              </div>
            </div>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>
<script type="text/babel">
import { iSelect , iOption } from 'iview'
import { getPartnerList , getWeekCalendarData , getCurrentWeek , getCurrentYearTotalWeek , getProjectList} from 'services/calendar-week/calendar-week'
export default{
  components: { iSelect , iOption },
  data() {
    return {
      sx: 0,
      sy: 0,
      ex: 0,
      ey: 0,
      userId: null,
      distance: 0, // 水平滚动的距离
      limitRight: 750-$(window).width(), //适配不同屏幕右边的空隙为0
      weeks: ['一','二','三','四','五','六','日'],
      projectList: [], // 总项目
      calendarIconState: true, //  周日历切换的icon
      currentProject: null, //当前选中的项目
      weeksContentInitLeft: 0,//周 滚动条
      weeksContentFactLeft: 0, // 周内容滚动
      isPortraitScroll: false, // 判断滚动的方向
      partnerRows: getPartnerList, //周历全部空数据
      partnerListInit: [], //包含单格内多于2个的总周历数据
      weekCalendarList: [], // 删除掉单格多于2个的总周历数据
      popupPartnerDetail: [], //弹窗显示当前单格内的总数
      currentDate: new Date(), //当前日期
      currentWeekNumber: getCurrentWeek(new Date()), //当前周(数字)
      currentWeekString: "", //当前周(字符串)
      currentYear: new Date().getFullYear(), //当前年
      //获取当前年份的总周数
      currentYearTotalWeek: getCurrentYearTotalWeek(this.currentYear),
    }
  },
  ready () {
    //处理数据
    this.weekToString();
    $('.ivu-select-dropdown').css('width', '80px')
    this.userId = !!this.$route.query.userId ? this.$route.query.userId : null;
    this.projectList = getProjectList(this.userId);
    this.currentProject = this.projectList.length > 0 ? this.projectList[0].projectId : null;

    this.weekCalendarList = getWeekCalendarData(this.currentProject, this.userId, new Date().getFullYear().toString() + this.currentWeekNumber.toString());
    this.replaceWeekData();//数据写入表格
    // 调用原生端的方法
    if (navigator.userAgent.indexOf('Android') > -1 || navigator.userAgent.indexOf('Adr') > -1) {
      if (window.android) {
        window.android.changeTitle('团队周历')
      }
    }
    //此处为IOS手机
    if (!!navigator.userAgent.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/)) {
      if (window.webkit) {
        window.webkit.messageHandlers.changeTitle.postMessage('团队周历')
      }
    }
  },
  methods:{
    changeProject () {//切换项目改变周历视图
      this.currentWeekNumber = getCurrentWeek(new Date());
      this.clearWeekData();
      // 从后台拿到实际的数据
      this.weekCalendarList = getWeekCalendarData(this.$refs.project.model, this.userId, new Date().getFullYear().toString() + this.currentWeekNumber.toString());
      this.replaceWeekData();
    },
    lastWeek () {//获取上一周
      this.clearWeekData();
      this.currentWeekNumber--;
      if (this.currentWeekNumber < 1) {
        this.currentYear--;
        this.currentWeekNumber = getCurrentYearTotalWeek(this.currentYear);
      }
      this.weekToString();
      this.yearWeek = this.currentYear.toString() + this.currentWeekString;
      // 从后台拿到实际的数据
      this.weekCalendarList = getWeekCalendarData(this.currentProject, this.userId, this.yearWeek);
      this.replaceWeekData();
    },
    nextWeek () {//获取下一周
      this.clearWeekData();
      this.currentWeekNumber++;
      if (this.currentWeekNumber > this.currentYearTotalWeek - 1){
        this.currentYear++;
        this.currentWeekNumber = 1;
        this.currentYearTotalWeek = getCurrentYearTotalWeek(this.currentYear)
      }
      this.weekToString();
      this.yearWeek = this.currentYear.toString() + this.currentWeekString;
      // 从后台拿到实际的数据
      this.weekCalendarList = getWeekCalendarData(this.currentProject, this.userId, this.yearWeek);
      this.replaceWeekData();
    },
    replaceWeekData () {// 单格少于3个的总周历数据写入表格
      let self = this
      let yearWeek = self.currentYear.toString() + self.currentWeekString;
      
      // 处理时间以便于对号入座
      if (self.weekCalendarList instanceof Array) {
        self.weekCalendarList.forEach(item1=>{
          item1.calendars = eval("(" + item1.calendars + ")")

          item1.calendars.forEach(item2 => {
            self.partnerRows[item1.day].hours[Number(item2.time.substr(0, 2))].calendars.push(item2)
          })
        })


        // 拷贝一份，一份删除单格多于2的作为展示,另一份未删除单格多于2个作为弹窗展示
        $.extend(true, self.partnerListInit, self.partnerRows);
        self.partnerRows.forEach(item1 => {
          item1.hours.forEach(item2 => {
            if (item2.calendars.length > 2){
              item2.calendars = item2.calendars.slice(0, 2);
              item2.thanTwo = true;
            }
          })
        })
      }else {
        this.clearWeekData()
      }
    },
    clearWeekData () { //清空周数据(再进行'描点')
      this.partnerRows.forEach(item1 => {
        item1.hours.forEach(item2 => {
          item2.calendars = [];
          item2.thanTwo = false;
          item2.popup = false;
        })
      })
    },
    calendarDetailState (item, index){ //点击弹窗
      this.partner.forEach(items => {
        items.state = false;
      })
      item.state = !item.state;
    },
    horizontalTouchStart (e) { //水平滚动
      e = e || window.event;
      this.sx = e.touches[0].pageX;
      this.sy = e.touches[0].pageY;
    },
    horizontalTouchMove (e) { //水平滚动
      e = e || window.event;
      this.ex = e.touches[0].pageX;
      this.ey = e.touches[0].pageY;
      this.distance = this.ex - this.sx;
      if (Math.abs(this.distance) < 20 && Math.abs(this.ey - this.sy) > 30) {
        this.isPortraitScroll = true;
      }else{
        this.isPortraitScroll = false;
        this.weeksContentFactLeft = this.weeksContentInitLeft + this.distance;
        if (this.weeksContentFactLeft < -this.limitRight ){
            this.weeksContentFactLeft = -this.limitRight
        }
        if (this.weeksContentFactLeft > 0 ){
          this.weeksContentFactLeft = 0;
        }
      }
    },
    portraitTouchEnd () { //垂直滚动
      this.weeksContentInitLeft = this.weeksContentFactLeft;
    },
    partnerDetail (index1,index2){ //弹窗显示详细信息
      this.popupPartnerDetail = this.partnerListInit[index1].hours[index2];
      this.partnerRows.forEach(item => {
        item.hours.forEach(items => {
          items.popup = false;
        })
      });
      this.partnerRows[index1].hours[index2].popup = !this.partnerRows[index1].hours[index2].popup
    },
    weekToString () { //当前周数转为字符串
      if (this.currentWeekNumber < 10) {
        this.currentWeekString = "0" + this.currentWeekNumber.toString();
      }else{
        this.currentWeekString = this.currentWeekNumber.toString();
      }
    }
  }
}
</script>
<style lang="stylus" scoped>
@import '~assets/css/calendar/calendar.styl'

</style>