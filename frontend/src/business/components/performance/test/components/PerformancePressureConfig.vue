<template>
  <div v-loading="result.loading" class="pressure-config-container">
    <el-row>
      <el-col>
        <el-form :inline="true">
          <el-form-item :label="$t('load_test.select_resource_pool')">
            <el-select v-model="resourcePool" :disabled="isReadOnly" size="mini" @change="resourcePoolChange">
              <el-option
                v-for="item in resourcePools"
                :key="item.id"
                :label="item.name"
                :value="item.id">
              </el-option>
            </el-select>
          </el-form-item>
          <el-form-item :label="$t('load_test.serialize_threadgroups')">
            <el-switch v-model="serializeThreadGroups"/>
          </el-form-item>
          <br>
          <el-form-item :label="$t('load_test.autostop_threadgroups')">
            <el-switch v-model="autoStop"/>
          </el-form-item>
          <el-form-item :label="$t('load_test.reaches_duration')">
            <el-input-number
              :disabled="isReadOnly || !autoStop"
              v-model="autoStopDelay"
              :min="1"
              :max="9999"
              size="mini"/>
          </el-form-item>
          <el-form-item :label="$t('load_test.autostop_delay')"/>
        </el-form>
      </el-col>
    </el-row>
    <el-row>
      <el-col :span="12">
        <el-collapse v-model="activeNames" accordion>
          <el-collapse-item :name="index"
                            v-for="(threadGroup, index) in threadGroups.filter(th=>th.enabled === 'true' && th.deleted=='false')"
                            :key="index">
            <template slot="title">
              <el-row>
                <el-col :span="14">
                  <el-tooltip :content="threadGroup.attributes.testname" placement="top">
                    <div style="padding-right:20px; font-size: 16px;" class="wordwrap">
                      {{ threadGroup.attributes.testname }}
                    </div>
                  </el-tooltip>
                </el-col>
                <el-col :span="10">
                  <el-tag type="primary" size="mini" v-if="threadGroup.threadType === 'DURATION'">
                    {{ $t('load_test.thread_num') }}{{ threadGroup.threadNumber }},
                    {{ $t('load_test.duration') }}: {{ threadGroup.duration }} {{ getUnitLabel(threadGroup) }}
                  </el-tag>
                  <el-tag type="primary" size="mini" v-if="threadGroup.threadType === 'ITERATION'">
                    {{ $t('load_test.thread_num') }} {{ threadGroup.threadNumber }},
                    {{ $t('load_test.iterate_num') }} {{ threadGroup.iterateNum }}
                  </el-tag>
                </el-col>
              </el-row>
            </template>
            <el-form :inline="true">
              <el-form-item :label="$t('load_test.thread_num')">
                <el-input-number
                  :disabled="isReadOnly"
                  v-model="threadGroup.threadNumber"
                  @change="calculateTotalChart()"
                  :min="resourcePoolResourceLength"
                  :max="maxThreadNumbers"
                  size="mini"/>
              </el-form-item>
              <br>
              <el-form-item>
                <el-radio-group v-model="threadGroup.threadType" @change="calculateTotalChart()">
                  <el-radio label="DURATION">{{ $t('load_test.by_duration') }}</el-radio>
                  <el-radio label="ITERATION">{{ $t('load_test.by_iteration') }}</el-radio>
                </el-radio-group>
              </el-form-item>
              <br>
              <div v-if="threadGroup.threadType === 'DURATION'">
                <el-form-item :label="$t('load_test.duration')">
                  <el-input-number
                    :disabled="isReadOnly"
                    v-model="threadGroup.duration"
                    :min="1"
                    :max="9999"
                    @change="calculateTotalChart()"
                    size="mini"/>
                </el-form-item>
                <el-form-item>
                  <el-radio-group v-model="threadGroup.unit">
                    <el-radio label="S">{{ $t('schedule.cron.seconds') }}</el-radio>
                    <el-radio label="M">{{ $t('schedule.cron.minutes') }}</el-radio>
                    <el-radio label="H">{{ $t('schedule.cron.hours') }}</el-radio>
                  </el-radio-group>
                </el-form-item>
                <br>
                <el-form-item :label="$t('load_test.rps_limit')">
                  <el-switch v-model="threadGroup.rpsLimitEnable" @change="calculateTotalChart()"/>
                  &nbsp;
                  <el-input-number
                    :disabled="isReadOnly || !threadGroup.rpsLimitEnable"
                    v-model="threadGroup.rpsLimit"
                    @change="calculateTotalChart()"
                    :min="1"
                    :max="99999"
                    size="mini"/>
                </el-form-item>
                <br>
                <div v-if="threadGroup.tgType === 'com.blazemeter.jmeter.threads.concurrency.ConcurrencyThreadGroup'">
                  <el-form-item :label="$t('load_test.ramp_up_time_within')">
                    <el-input-number
                      :disabled="isReadOnly"
                      :min="1"
                      :max="threadGroup.duration"
                      v-model="threadGroup.rampUpTime"
                      @change="calculateTotalChart()"
                      size="mini"/>
                  </el-form-item>
                  <el-form-item :label="$t('load_test.ramp_up_time_minutes', [getUnitLabel(threadGroup)])">
                    <el-input-number
                      :disabled="isReadOnly"
                      :min="1"
                      :max="Math.min(threadGroup.threadNumber, threadGroup.rampUpTime)"
                      v-model="threadGroup.step"
                      @change="calculateTotalChart()"
                      size="mini"/>
                  </el-form-item>
                  <el-form-item :label="$t('load_test.ramp_up_time_times')"/>
                </div>

                <div v-if="threadGroup.tgType === 'ThreadGroup'">
                  <el-form-item :label="$t('load_test.ramp_up_time_within')">
                    <el-input-number
                      :disabled="isReadOnly"
                      :min="1"
                      :max="threadGroup.duration"
                      v-model="threadGroup.rampUpTime"
                      @change="calculateTotalChart()"
                      size="mini"/>
                  </el-form-item>
                  <el-form-item :label="$t('load_test.ramp_up_time_seconds', [getUnitLabel(threadGroup)])"/>
                </div>

              </div>
              <div v-if="threadGroup.threadType === 'ITERATION'">
                <el-form-item :label="$t('load_test.iterate_num')">
                  <el-input-number
                    :disabled="isReadOnly"
                    v-model="threadGroup.iterateNum"
                    :min="1"
                    :max="9999999"
                    @change="calculateTotalChart()"
                    size="mini"/>
                </el-form-item>
                <br>
                <el-form-item :label="$t('load_test.rps_limit')">
                  <el-switch v-model="threadGroup.rpsLimitEnable" @change="calculateTotalChart()"/>
                  &nbsp;
                  <el-input-number
                    :disabled="isReadOnly || !threadGroup.rpsLimitEnable"
                    v-model="threadGroup.rpsLimit"
                    :min="1"
                    :max="99999"
                    size="mini"/>
                </el-form-item>
                <br>
                <el-form-item :label="$t('load_test.ramp_up_time_within')">
                  <el-input-number
                    :disabled="isReadOnly"
                    :min="1"
                    v-model="threadGroup.iterateRampUp"
                    size="mini"/>
                </el-form-item>
                <el-form-item :label="$t('load_test.ramp_up_time_seconds', [getUnitLabel(threadGroup)])"/>
              </div>
            </el-form>
          </el-collapse-item>
        </el-collapse>
      </el-col>
      <el-col :span="12">
        <div class="title">{{ $t('load_test.pressure_prediction_chart') }}</div>
        <ms-chart class="chart-container" ref="chart1" :options="options" :autoresize="true"></ms-chart>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import echarts from "echarts";
import MsChart from "@/business/components/common/chart/MsChart";
import {findThreadGroup} from "@/business/components/performance/test/model/ThreadGroup";
import {hasPermission} from "@/common/js/utils";

const HANDLER = "handler";
const THREAD_GROUP_TYPE = "tgType";
const SERIALIZE_THREAD_GROUPS = "serializeThreadGroups";
const AUTO_STOP = "autoStop";
const AUTO_STOP_DELAY = "autoStopDelay";
const TARGET_LEVEL = "TargetLevel";
const RAMP_UP = "RampUp";
const ITERATE_RAMP_UP = "iterateRampUpTime";
const STEPS = "Steps";
const DURATION = "duration";
const UNIT = "unit";
const RPS_LIMIT = "rpsLimit";
const RPS_LIMIT_ENABLE = "rpsLimitEnable";
const HOLD = "Hold";
const THREAD_TYPE = "threadType";
const ITERATE_NUM = "iterateNum";
const ENABLED = "enabled";
const DELETED = "deleted";

const hexToRgba = function (hex, opacity) {
  return 'rgba(' + parseInt('0x' + hex.slice(1, 3)) + ',' + parseInt('0x' + hex.slice(3, 5)) + ','
    + parseInt('0x' + hex.slice(5, 7)) + ',' + opacity + ')';
};
const hexToRgb = function (hex) {
  return 'rgb(' + parseInt('0x' + hex.slice(1, 3)) + ',' + parseInt('0x' + hex.slice(3, 5))
    + ',' + parseInt('0x' + hex.slice(5, 7)) + ')';
};

export default {
  name: "PerformancePressureConfig",
  components: {MsChart},
  props: {
    test: {
      type: Object
    },
    testId: {
      type: String
    }
  },
  data() {
    return {
      result: {},
      threadNumber: 0,
      duration: 0,
      rampUpTime: 0,
      step: 0,
      rpsLimit: 0,
      rpsLimitEnable: false,
      options: {},
      resourcePool: null,
      resourcePools: [],
      activeNames: ["0"],
      threadGroups: [],
      resourcePoolResourceLength: 1,
      maxThreadNumbers: 5000,
      serializeThreadGroups: false,
      autoStop: false,
      autoStopDelay: 30,
      isReadOnly: false,
    };
  },
  mounted() {
    if (this.testId) {
      this.getJmxContent();
    } else {
      this.calculateTotalChart();
    }
    this.resourcePool = this.test.testResourcePoolId;
    this.getResourcePools();
    this.isReadOnly = !hasPermission('PROJECT_PERFORMANCE_TEST:READ+EDIT');
  },
  watch: {
    test(n) {
      this.resourcePool = n.testResourcePoolId;
    },
    testId() {
      if (this.testId) {
        this.getJmxContent();
      } else {
        this.calculateTotalChart();
      }
      this.getResourcePools();
    },
  },
  methods: {
    getResourcePools() {
      this.result = this.$get('/testresourcepool/list/quota/valid', response => {
        this.resourcePools = response.data;
        // 如果当前的资源池无效 设置 null
        if (response.data.filter(p => p.id === this.resourcePool).length === 0) {
          this.resourcePool = null;
        }

        this.resourcePoolChange();
      });
    },
    getLoadConfig() {
      this.$get('/performance/get-load-config/' + this.testId, (response) => {
        if (response.data) {
          let data = JSON.parse(response.data);
          for (let i = 0; i < data.length; i++) {
            let d = data[i];
            d.forEach(item => {
              switch (item.key) {
                case TARGET_LEVEL:
                  this.threadGroups[i].threadNumber = item.value;
                  break;
                case RAMP_UP:
                  this.threadGroups[i].rampUpTime = item.value;
                  break;
                case ITERATE_RAMP_UP:
                  this.threadGroups[i].iterateRampUp = item.value;
                  break;
                case DURATION:
                  this.threadGroups[i].duration = item.value;
                  break;
                case UNIT:
                  this.threadGroups[i].unit = item.value;
                  break;
                case STEPS:
                  this.threadGroups[i].step = item.value;
                  break;
                case RPS_LIMIT:
                  this.threadGroups[i].rpsLimit = item.value;
                  break;
                case RPS_LIMIT_ENABLE:
                  this.threadGroups[i].rpsLimitEnable = item.value;
                  break;
                case THREAD_TYPE:
                  this.threadGroups[i].threadType = item.value;
                  break;
                case ITERATE_NUM:
                  this.threadGroups[i].iterateNum = item.value;
                  break;
                case ENABLED:
                  this.threadGroups[i].enabled = item.value;
                  break;
                case DELETED:
                  this.threadGroups[i].deleted = item.value;
                  break;
                case HANDLER:
                  this.threadGroups[i].handler = item.value;
                  break;
                case THREAD_GROUP_TYPE:
                  this.threadGroups[i].tgType = item.value;
                  break;
                case SERIALIZE_THREAD_GROUPS:
                  this.serializeThreadGroups = item.value;// 所有的线程组值一样
                  break;
                case AUTO_STOP:
                  this.autoStop = item.value;// 所有的线程组值一样
                  break;
                case AUTO_STOP_DELAY:
                  this.autoStopDelay = item.value;// 所有的线程组值一样
                  break;
                default:
                  break;
              }
              //
              this.$set(this.threadGroups[i], "unit", this.threadGroups[i].unit || 'S');
              this.$set(this.threadGroups[i], "threadType", this.threadGroups[i].threadType || 'DURATION');
              this.$set(this.threadGroups[i], "iterateNum", this.threadGroups[i].iterateNum || 1);
              this.$set(this.threadGroups[i], "iterateRampUp", this.threadGroups[i].iterateRampUp || 10);
              this.$set(this.threadGroups[i], "enabled", this.threadGroups[i].enabled || 'true');
              this.$set(this.threadGroups[i], "deleted", this.threadGroups[i].deleted || 'false');
            });
          }
          this.calculateTotalChart();
        }
      });
    },
    getJmxContent() {
      if (this.testId) {
        let threadGroups = [];
        this.$get('/performance/get-jmx-content/' + this.testId, (response) => {
          response.data.forEach(d => {
            threadGroups = threadGroups.concat(findThreadGroup(d.jmx, d.name));
            threadGroups.forEach(tg => {
              tg.options = {};
            });
          });
          this.threadGroups = threadGroups;
          this.$emit('fileChange', threadGroups);
          this.getLoadConfig();
        });
      }
    },
    resourcePoolChange() {
      let result = this.resourcePools.filter(p => p.id === this.resourcePool);
      if (result.length === 1) {
        let threadNumber = 0;
        result[0].resources.forEach(resource => {
          threadNumber += JSON.parse(resource.configuration).maxConcurrency;
        });
        this.$set(this, 'maxThreadNumbers', threadNumber);
        this.threadGroups.forEach(tg => {
          if (tg.threadNumber > threadNumber) {
            this.$set(tg, "threadNumber", threadNumber);
          }
        });
        this.calculateTotalChart();
      }
    },
    calculateTotalChart() {
      let handler = this;
      if (handler.duration < handler.rampUpTime) {
        handler.rampUpTime = handler.duration;
      }
      if (handler.rampUpTime < handler.step) {
        handler.step = handler.rampUpTime;
      }
      // 线程数不能小于资源池节点的数量
      let resourcePool = this.resourcePools.filter(v => v.id === this.resourcePool)[0];
      if (resourcePool) {
        this.resourcePoolResourceLength = resourcePool.resources.length;
      }
      let color = ['#60acfc', '#32d3eb', '#5bc49f', '#feb64d', '#ff7c7c', '#9287e7', '#ca8622', '#bda29a', '#6e7074', '#546570', '#c4ccd3'];
      handler.options = {
        color: color,
        xAxis: {
          type: 'category',
          boundaryGap: false,
          data: []
        },
        yAxis: {
          type: 'value'
        },
        tooltip: {
          trigger: 'axis',
        },
        series: []
      };

      for (let i = 0; i < handler.threadGroups.length; i++) {
        if (handler.threadGroups[i].enabled === 'false' ||
          handler.threadGroups[i].deleted === 'true' ||
          handler.threadGroups[i].threadType === 'ITERATION') {
          continue;
        }
        let seriesData = {
          name: handler.threadGroups[i].attributes.testname,
          data: [],
          type: 'line',
          smooth: false,
          symbolSize: 5,
          showSymbol: false,
          lineStyle: {
            normal: {
              width: 1
            }
          },
          areaStyle: {
            normal: {
              color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [{
                offset: 0,
                color: hexToRgba(color[i % color.length], 0.3),
              }, {
                offset: 0.8,
                color: hexToRgba(color[i % color.length], 0),
              }], false),
              shadowColor: 'rgba(0, 0, 0, 0.1)',
              shadowBlur: 10
            }
          },
          itemStyle: {
            normal: {
              color: hexToRgb(color[i % color.length]),
              borderColor: 'rgba(137,189,2,0.27)',
              borderWidth: 12
            }
          },
        };

        let tg = handler.threadGroups[i];

        let timePeriod = Math.floor(tg.rampUpTime / tg.step);
        let timeInc = timePeriod;

        let threadPeriod = Math.floor(tg.threadNumber / tg.step);
        let threadInc1 = Math.floor(tg.threadNumber / tg.step);
        let threadInc2 = Math.ceil(tg.threadNumber / tg.step);
        let inc2count = tg.threadNumber - tg.step * threadInc1;
        for (let j = 0; j <= tg.duration; j++) {
          // x 轴
          let xAxis = handler.options.xAxis.data;
          if (xAxis.indexOf(j) < 0) {
            xAxis.push(j);
          }
          if (tg.tgType === 'ThreadGroup') {
            seriesData.step = undefined;

            if (j === 0) {
              seriesData.data.push([0, 0]);
            }
            if (j >= tg.rampUpTime) {
              xAxis.push(tg.duration);

              seriesData.data.push([j, tg.threadNumber]);
              seriesData.data.push([tg.duration, tg.threadNumber]);
              break;
            }
          } else {
            seriesData.step = 'start';
            if (j > timePeriod) {
              timePeriod += timeInc;
              if (inc2count > 0) {
                threadPeriod = threadPeriod + threadInc2;
                inc2count--;
              } else {
                threadPeriod = threadPeriod + threadInc1;
              }
              if (threadPeriod > tg.threadNumber) {
                threadPeriod = tg.threadNumber;
                // 预热结束
                xAxis.push(tg.duration);
                seriesData.data.push([tg.duration, threadPeriod]);
                break;
              }
            }
            seriesData.data.push([j, threadPeriod]);
          }
        }
        handler.options.series.push(seriesData);
      }
    },
    calculateChart(threadGroup) {
      let handler = this;
      if (threadGroup) {
        handler = threadGroup;
      }
      if (handler.duration < handler.rampUpTime) {
        handler.rampUpTime = handler.duration;
      }
      if (handler.rampUpTime < handler.step) {
        handler.step = handler.rampUpTime;
      }
      // 线程数不能小于资源池节点的数量
      let resourcePool = this.resourcePools.filter(v => v.id === this.resourcePool)[0];
      if (resourcePool) {
        this.resourcePoolResourceLength = resourcePool.resources.length;
      }
      handler.options = {
        xAxis: {
          type: 'category',
          boundaryGap: false,
          data: []
        },
        yAxis: {
          type: 'value'
        },
        tooltip: {
          trigger: 'axis',
          formatter: '{a}: {c0}',
          axisPointer: {
            lineStyle: {
              color: '#57617B'
            }
          }
        },
        series: [{
          name: 'User',
          data: [],
          type: 'line',
          step: 'start',
          smooth: false,
          symbolSize: 5,
          showSymbol: false,
          lineStyle: {
            normal: {
              width: 1
            }
          },
          areaStyle: {
            normal: {
              color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [{
                offset: 0,
                color: 'rgba(137, 189, 27, 0.3)'
              }, {
                offset: 0.8,
                color: 'rgba(137, 189, 27, 0)'
              }], false),
              shadowColor: 'rgba(0, 0, 0, 0.1)',
              shadowBlur: 10
            }
          },
          itemStyle: {
            normal: {
              color: 'rgb(137,189,27)',
              borderColor: 'rgba(137,189,2,0.27)',
              borderWidth: 12
            }
          },
        }]
      };
      let timePeriod = Math.floor(handler.rampUpTime / handler.step);
      let timeInc = timePeriod;

      let threadPeriod = Math.floor(handler.threadNumber / handler.step);
      let threadInc1 = Math.floor(handler.threadNumber / handler.step);
      let threadInc2 = Math.ceil(handler.threadNumber / handler.step);
      let inc2count = handler.threadNumber - handler.step * threadInc1;
      for (let i = 0; i <= handler.duration; i++) {
        // x 轴
        handler.options.xAxis.data.push(i);

        if (handler.tgType === 'ThreadGroup') {
          handler.options.series[0].step = undefined;

          if (i === 0) {
            handler.options.series[0].data.push([0, 0]);
          }
          if (i >= handler.rampUpTime) {
            handler.options.xAxis.data.push(handler.duration);

            handler.options.series[0].data.push([i, handler.threadNumber]);
            handler.options.series[0].data.push([handler.duration, handler.threadNumber]);
            break;
          }
        } else {
          handler.options.series[0].step = 'start';
          if (i > timePeriod) {
            timePeriod += timeInc;
            if (inc2count > 0) {
              threadPeriod = threadPeriod + threadInc2;
              inc2count--;
            } else {
              threadPeriod = threadPeriod + threadInc1;
            }
            if (threadPeriod > handler.threadNumber) {
              threadPeriod = handler.threadNumber;
              handler.options.xAxis.data.push(handler.duration);
              handler.options.series[0].data.push([handler.duration, handler.threadNumber]);
              break;
            }
            handler.options.series[0].data.push([i, threadPeriod]);
          } else {
            handler.options.series[0].data.push([i, threadPeriod]);
          }
        }
      }
      this.calculateTotalChart();
    },
    validConfig() {
      if (!this.resourcePool) {
        this.$warning(this.$t('load_test.resource_pool_is_null'));
        // 资源池为空，设置参数为资源池所在Tab的name
        this.$emit('changeActive', '1');
        return false;
      }

      for (let i = 0; i < this.threadGroups.length; i++) {
        if (!this.threadGroups[i].threadNumber || !this.threadGroups[i].duration
          || !this.threadGroups[i].rampUpTime || !this.threadGroups[i].step || !this.threadGroups[i].iterateNum) {
          this.$warning(this.$t('load_test.pressure_config_params_is_empty'));
          this.$emit('changeActive', '1');
          return false;
        }

        if (this.threadGroups[i].rpsLimitEnable && !this.threadGroups[i].rpsLimit) {
          this.$warning(this.$t('load_test.pressure_config_params_is_empty'));
          this.$emit('changeActive', '1');
          return false;
        }
      }

      return true;
    },
    getUnitLabel(tg) {
      if (tg.unit === 'S') {
        return this.$t('schedule.cron.seconds');
      }
      if (tg.unit === 'M') {
        return this.$t('schedule.cron.minutes');
      }
      if (tg.unit === 'H') {
        return this.$t('schedule.cron.hours');
      }
      return this.$t('schedule.cron.seconds');
    },
    convertProperty() {
      /// todo：下面4个属性是jmeter ConcurrencyThreadGroup plugin的属性，这种硬编码不太好吧，在哪能转换这种属性？
      let result = [];

      // 再组织数据
      for (let i = 0; i < this.threadGroups.length; i++) {
        result.push([
          {key: HANDLER, value: this.threadGroups[i].handler},
          {key: TARGET_LEVEL, value: this.threadGroups[i].threadNumber},
          {key: RAMP_UP, value: this.threadGroups[i].rampUpTime},
          {key: STEPS, value: this.threadGroups[i].step},
          {key: DURATION, value: this.threadGroups[i].duration, unit: this.threadGroups[i].unit},
          {key: UNIT, value: this.threadGroups[i].unit},
          {key: RPS_LIMIT, value: this.threadGroups[i].rpsLimit},
          {key: RPS_LIMIT_ENABLE, value: this.threadGroups[i].rpsLimitEnable},
          {key: HOLD, value: this.threadGroups[i].duration - this.threadGroups[i].rampUpTime},
          {key: THREAD_TYPE, value: this.threadGroups[i].threadType},
          {key: ITERATE_NUM, value: this.threadGroups[i].iterateNum},
          {key: ITERATE_RAMP_UP, value: this.threadGroups[i].iterateRampUp},
          {key: ENABLED, value: this.threadGroups[i].enabled},
          {key: DELETED, value: this.threadGroups[i].deleted},
          {key: THREAD_GROUP_TYPE, value: this.threadGroups[i].tgType},
          {key: SERIALIZE_THREAD_GROUPS, value: this.serializeThreadGroups},
          {key: AUTO_STOP, value: this.autoStop},
          {key: AUTO_STOP_DELAY, value: this.autoStopDelay},
        ]);
      }

      return result;
    }
  }
};
</script>


<style scoped>
.pressure-config-container .el-input {
  width: 130px;

}

.pressure-config-container .config-form-label {
  width: 130px;
}

.pressure-config-container .input-bottom-border input {
  border: 0;
  border-bottom: 1px solid #DCDFE6;
}

/deep/ .el-collapse-item__content {
  padding-left: 10px;
  padding-bottom: 5px;
  border-left-width: 8px;
  border-left-style: solid;
  border-left-color: #F5F7FA;
  border-top-left-radius: 3px;
  border-bottom-left-radius: 3px;
}

.chart-container {
  width: 100%;
  height: 300px;
}

.el-form-item {
  margin-top: 5px;
  margin-bottom: 5px;
}

.el-col .el-form {
  margin-top: 5px;
  text-align: left;
}

.el-col {
  margin-top: 5px;
  text-align: left;
}

.title {
  margin-left: 60px;
}

.wordwrap {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>
