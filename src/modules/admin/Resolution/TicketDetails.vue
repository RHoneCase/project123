<template>
<div id="holder">
  <div class="title" style="margin-top: 25px;">
    <label class="text-primary action-link" @click="redirect('/tickets')"><i class="fas fa-arrow-left"></i> <b> back </b>to previous</label>
  </div>
  <span v-if="data.length !== 0">
    <span id="title"><b>{{title}}</b>#{{data.id}}</span>
      <br><span>{{status}} {{ timeIntervalRes + ' ago'}} by {{data.account_id}}</span>
    </span>
   <div class="row" >
    <div class="col-10" id="detail">
      <form>
        <div class="form-group">
          <label for="text"><b>Title</b></label>
          <input type="text" class="form-control" v-model="title" id="text">
        </div>
        <div class="form-group">
          <label for="pwd"><b>Details</b></label>
          <textarea type="password" v-model="detail" class="form-control"  id="pwd"/>
        </div>
        <span><b>Image attachments</b></span><br>

        <multiple-img-uploader  v-if="data.length !== 0" :imageList="imageList" :isEditableProp="editable"/>
        <button type="button" class="btn btn-primary mb-5" @click="update()" id="update">Update</button>

      </form>
    </div>
    <div class="col-2" id="uneditableDetail">
      <div>
        <ticket-type v-if="data.length !== 1" :isEditable="{isEditable: user.userID === data.account_id, typeResult: data.type}"/>
      <hr>
      <span>Assignee</span>
      <p @click="showAssignees()" style="color:grey; cursor: pointer;">{{ assignee ? assignee : 'no assigned resolver'}}</p>
      <assignees ref="assign"></assignees>
      <hr>
      <span>Status</span>
      <select :required="true" class="form-control" v-model="data.status">
        <option v-for="(option, index) in options" :selected="data.status" v-bind:key="index">{{option.name}}</option>
      </select>
      </div>
    </div>
  </div>
</div>

</template>

<script>
import ROUTER from 'src/router'
import MultipleImgUploader from './MultipleImgUploader'
import AUTH from 'src/services/auth'
import TimeInterval from './TimeInterval.js'
import TicketType from './TicketTypes.js'
import TicketTypeComp from './TicketType.vue'

export default {
  created() {
    this.retrieveItem(this.$route.params.id)
  },
  data() {
    return {
      user: AUTH.user,
      imageList: [],
      data: [],
      timeInterval: TimeInterval,
      title: null,
      detail: null,
      status: null,
      ticketType: TicketType.types,
      timeIntervalRes: null,
      editable: true,
      parameter: {
        image: null
      },
      assignee: null,
      options: [
        {
          name: 'open'
        },
        {
          name: 'pending'
        },
        {
          name: 'closed'
        }
      ]
    }
  },
  components: {
    'multiple-img-uploader': MultipleImgUploader,
    'ticket-type': TicketTypeComp,
    'assignees': require('modules/admin/Resolution/Assignees.vue')
  },
  methods: {
    showAssignees(){
      this.$refs.assign.retrieveAssignees()
    },
    update() {
      let images = this.imageList.join(' ')
      let parameter = {
        id: this.$route.params.id,
        title: this.title,
        content: this.title,
        images: images,
        assigned_to: this.assignee,
        status: this.data.status
      }
      // console.log(parameter.status, 'oo')
      $('#loading').css({display: 'block'})
      this.APIRequest('tickets/update', parameter).then(response => {
        $('#loading').css({display: 'none'})
        if(response.data === true) {
          this.retrieveItem(this.$route.params.id)
        }
      })
    },
    redirect(path){
      ROUTER.push(path)
    },
    retrieveItem(id){
      let parameter = {
        condition: [{
          value: id,
          column: 'id',
          clause: '='
        }]
      }
      $('#loading').css({display: 'block'})
      this.APIRequest('tickets/retrieve', parameter).then(response => {
        $('#loading').css({display: 'none'})
        if(response.data.length > 0){
          this.data = response.data[0]
          this.timeIntervalRes = this.getticketTimePassed(this.data.created_at)
          if(this.data.images !== null) {
            console.log(this.data.images)
            this.imageList = this.data.images.split(' ')
          }
          this.editable = this.user.userID === this.data.account_id
          this.statusSelected = this.data.status.toLowerCase()
          switch(this.data.status.toLowerCase()){
            case 'open':
              this.status = 'opened'
              break
            case 'pending':
              this.status = 'moved to pending status'
              break
            case 'close':
              this.status = 'closed'
              break
          }
          this.title = this.data.content[0] === '{' ? JSON.parse(this.data.content).title : this.data.content
          this.detail = this.data.content[0] === '{' ? JSON.parse(this.data.content).detail : this.data.content
        }else{
          this.data = []
        }
      })
    },
    getticketTimePassed(time){
      var ticketCreation = new Date(time)
      var current = new Date()
      var difference = (current - ticketCreation)
      let result = this.timeInterval.getticketTimePassed(difference)
      let timePassedFormat = ''
      console.log(difference, result)
      if(result.length === 1) {
        timePassedFormat += result[0].interval + ' ' + result[0].unit + (result[0].interval > 1 ? 's' : '')
        return timePassedFormat
      }
      if(result.length === 2) {
        timePassedFormat += result[0].interval + ' ' + result[0].unit + result[0].unit > 1 ? 's' : ''
        timePassedFormat += 'and' + ' ' + result[1].interval + ' ' + result[1].unit + result[1].unit > 1 ? 's' : ''
        return timePassedFormat
      }
      result.forEach(el => {
        if(result.indexOf(el) === 0) {
          timePassedFormat += el.interval + ' ' + el.unit + el.unit > 1 ? 's' : ''
        }
        if(result.indexOf(el) === result.length - 1) {
          timePassedFormat += 'and' + ' ' + el.interval + ' ' + el.unit + el.unit > 1 ? 's' : ''
        }
        timePassedFormat += ',' + ' ' + el.interval + ' ' + el.unit + el.unit > 1 ? 's' : ''
      })
      return timePassedFormat
    }
  }
}
</script>
<style lang="scss" scoped>
@import "~assets/style/colors.scss";
#holder {
  margin-top: 50px;
  width: 98%;
}
#title {
  font-size: 24px;
}
textarea{
  min-height: 200px;
}
form{
  margin-top: 25px;
}
#detail input{
  min-height: 75px;
}

#uneditableDetail div{
  margin-top: 50px;
}
#uneditableDetail #ticketType {
  background-color: red;
  border:none;
  border-radius: 20px;
  min-width: 50px;
  color: white;
}
#update{
  float: right;
}
</style>
