<template>
  <table
    :class="{'el-calendar-table': true, 'is-range': isInRange}"
    cellspacing="0"
    cellpadding="0">
    <thead>
      <th v-for="day in weekDays" :key="day">{{day}}</th>
    </thead>
    <tbody>
      <tr
        v-for="(row, index) in rows"
        :class="{
          'el-calendar-table__row': true,
          'el-calendar-table__row--hide-border': index === 0 && hideHeader
        }"
        :key="index"
      >
        <td
          v-for="(cell, key) in row"
          :key="key"
          :class="getCellClass(cell)"
          @click="pickDay(cell)"
        >
          <el-popover
            v-if="taskDay.indexOf(getFormateDate(cell.text, cell.type)) != -1 "
            placement="right-end"
            title="标题"
            width="400"
            trigger="hover"
            @show="initPage(getFormateDate(cell.text, cell.type))"
          >
            <div>
              <div v-for="(item, index) in localTaskDate[getFormateDate(cell.text, cell.type)]" :key="index">
                <div v-if="(index+1) == pageObj[getFormateDate(cell.text, cell.type)].current">
                  {{item.text}}
                </div>
              </div>
              <div>
                <el-pagination
                  :current-page.sync="pageObj[getFormateDate(cell.text, cell.type)].current"
                  :page-size="1"
                  layout="total, prev, pager, next, jumper"
                  :total="pageObj[getFormateDate(cell.text, cell.type)].total">
                </el-pagination>
              </div>
            </div>
            <div class="el-calendar-day" slot="reference">
              <span>{{cell.text}}</span>
            </div>
          </el-popover>
          <div v-else class="el-calendar-day">
            <span>{{cell.text}}</span>
          </div>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script>
import fecha from './utils/date';
import { range as rangeArr, getFirstDayOfMonth, getPrevMonthLastDays, getMonthDays, getI18nSettings, validateRangeInOneMonth } from './utils/date-util';

export default {
  props: {
    selectedDay: String, // formated date yyyy-MM-dd
    range: {
      type: Array,
      validator(val) {
        if (!(val && val.length)) return true;
        const [start, end] = val;
        return validateRangeInOneMonth(start, end);
      }
    },
    date: Date,
    hideHeader: Boolean,
    firstDayOfWeek: Number
  },

  inject: ['elCalendar'],

  data() {
    return {
      WEEK_DAYS: getI18nSettings().dayNames,
      taskDate: [
        {
          text: '任务1',
          date: '2021-05-14'
        },
        {
          text: '任务2',
          date: '2021-05-21'
        },
        {
          text: '任务3',
          date: '2021-05-14'
        },
        {
          text: '任务4',
          date: '2021-05-21'
        },
      ],
      localTaskDate: {},
      taskDay: [],
      pageObj: {}
    };
  },
  watch: {
    taskDate: {
      handler(newVal, oldVal) {
        let obj = {}
        let arr = []
        let obj1 = {}
        this.taskDate.forEach((item, index) => {
          if (obj[item.date]) {
            obj[item.date].push(item)
            obj1[item.date].total ++
          } else {
            obj[item.date] = []
            obj1[item.date] = {}
            obj1[item.date].total = 1
            obj1[item.date].current = 1
            obj[item.date].push(item)
          }
          if(arr.indexOf(item.date) == -1) {
            arr.push(item.date)
          }
        })
        this.localTaskDate = obj
        this.taskDay = arr
        this.pageObj = obj1
        console.log(this.localTaskDate, this.taskDay, this.pageObj)
      },
      immediate: true
    }
  },

  methods: {
    initPage(item) {
      this.pageObj[item].current = 1
    },
    // handleCurrentChange(val, item) {
    //   this.pageObj[item].current = val
    // },
    toNestedArr(days) {
      return rangeArr(days.length / 7).map((_, index) => {
        const start = index * 7;
        return days.slice(start, start + 7);
      });
    },

    getFormateDate(day, type) {
      if (!day || ['prev', 'current', 'next'].indexOf(type) === -1) {
        throw new Error('invalid day or type');
      }
      let prefix = this.curMonthDatePrefix;
      if (type === 'prev') {
        prefix = this.prevMonthDatePrefix;
      } else if (type === 'next') {
        prefix = this.nextMonthDatePrefix;
      }
      day = `00${day}`.slice(-2);
      return `${prefix}-${day}`;
    },

    getCellClass({ text, type}) {
      const classes = [type];
      if (type === 'current') {
        const date = this.getFormateDate(text, type);
        if (date === this.selectedDay) {
          classes.push('is-selected');
        }
        if (date === this.formatedToday) {
          classes.push('is-today');
        }
      }
      return classes;
    },

    pickDay({ text, type }) {
      const date = this.getFormateDate(text, type);
      this.$emit('pick', date);
    },

    cellRenderProxy({ text, type }) {
      let render = this.elCalendar.$scopedSlots.dateCell;
      if (!render) return <span>{ text }</span>;

      const day = this.getFormateDate(text, type);
      const date = new Date(day);
      const data = {
        isSelected: this.selectedDay === day,
        type: `${type}-month`,
        day
      };
      return render({ date, data });
    }
  },

  computed: {
    prevMonthDatePrefix() {
      const temp = new Date(this.date.getTime());
      temp.setDate(0);
      return fecha.format(temp, 'yyyy-MM');
    },

    curMonthDatePrefix() {
      return fecha.format(this.date, 'yyyy-MM');
    },

    nextMonthDatePrefix() {
      const temp = new Date(this.date.getFullYear(), this.date.getMonth() + 1, 1);
      return fecha.format(temp, 'yyyy-MM');
    },

    formatedToday() {
      return this.elCalendar.formatedToday;
    },

    isInRange() {
      return this.range && this.range.length;
    },

    rows() {
      let days = [];
      // if range exists, should render days in range.
      if (this.isInRange) {
        const [start, end] = this.range;
        const currentMonthRange = rangeArr(end.getDate() - start.getDate() + 1).map((_, index) => ({
          text: start.getDate() + index,
          type: 'current'
        }));
        let remaining = currentMonthRange.length % 7;
        remaining = remaining === 0 ? 0 : 7 - remaining;
        const nextMonthRange = rangeArr(remaining).map((_, index) => ({
          text: index + 1,
          type: 'next'
        }));
        days = currentMonthRange.concat(nextMonthRange);
      } else {
        const date = this.date;
        let firstDay = getFirstDayOfMonth(date);
        firstDay = firstDay === 0 ? 7 : firstDay;
        const firstDayOfWeek = typeof this.firstDayOfWeek === 'number' ? this.firstDayOfWeek : 1;
        const prevMonthDays = getPrevMonthLastDays(date, firstDay - firstDayOfWeek).map(day => ({
          text: day,
          type: 'prev'
        }));
        const currentMonthDays = getMonthDays(date).map(day => ({
          text: day,
          type: 'current'
        }));
        days = [...prevMonthDays, ...currentMonthDays];
        const nextMonthDays = rangeArr(42 - days.length).map((_, index) => ({
          text: index + 1,
          type: 'next'
        }));
        days = days.concat(nextMonthDays);
      }
      return this.toNestedArr(days);
    },

    weekDays() {
      const start = this.firstDayOfWeek;
      const { WEEK_DAYS } = this;

      if (typeof start !== 'number' || start === 0) {
        return WEEK_DAYS.slice();
      } else {
        return WEEK_DAYS.slice(start).concat(WEEK_DAYS.slice(0, start));
      }
    }
  },

  // render() {
  //   const thead = this.hideHeader ? null : (<thead>
  //     {
  //       this.weekDays.map(day => <th key={day}>{ day }</th>)
  //     }
  //   </thead>);
  //   return (
  //     <table
  //       class={{
  //         'el-calendar-table': true,
  //         'is-range': this.isInRange
  //       }}
  //       cellspacing="0"
  //       cellpadding="0">
  //       {
  //         thead
  //       }
  //       <tbody>
  //         {
  //           this.rows.map((row, index) => <tr
  //             class={{
  //               'el-calendar-table__row': true,
  //               'el-calendar-table__row--hide-border': index === 0 && this.hideHeader
  //             }}
  //             key={index}>
  //             {
  //               row.map((cell, key) => <td key={key}
  //                 class={ this.getCellClass(cell) }
  //                 onClick={this.pickDay.bind(this, cell)}>
  //                 <el-popover
  //                   placement="right-end"
  //                   title="标题"
  //                   width="200"
  //                   trigger="hover"
  //                   content="这是一段内容,这是一段内容,这是一段内容,这是一段内容。">
  //                   <div class="el-calendar-day" slot="reference">
  //                   {
  //                     this.cellRenderProxy(cell)
  //                   }
  //                   </div>
  //                 </el-popover>
  //               </td>)
  //             }
  //           </tr>)
  //         }
  //       </tbody>
  //     </table>);
  // }
};
</script>
