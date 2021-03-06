<template>
  <div class="we-steps">
    <Row class="we-steps-row-first">
      <Col
        span="2"
        class="we-steps-title"
      >Job状态：</Col>
      <Col
        span="22"
        class="we-steps-child-col"
      >
      <div
        class="we-steps-child"
        v-for="(child, index) in renderList"
        :key="index">
        <span class="we-steps-child-wrapper">
          <span
            class="we-steps-circle"
            :class="getClasses(child)"
            v-if="!child.isFinish">
          </span>
          <Icon
            v-else
            size="25"
            :color="getIconColor(child, index)"
            :type="getIconType(child)"></Icon>
          <Tooltip
            placement="right"
            theme="light"
            v-if="child.value==='FailedToGetResult'">
            <span
              style="cursor: pointer;"
              class="we-steps-label"
              :class="getClasses(child)">{{ child.label }}</span>
            <div
              slot="content"
              style="padding: 6px 10px;">
              <p style="font-weight: bold;line-height: 24px;">详情：</p>
              <p
                v-for="(p, index1) in hoverList"
                :key="index1"
                :style="{'color': p.includes('失败') ? 'red' : '#67c23a'}"
                style="line-height: 24px;">{{ p }}</p>
            </div>
          </Tooltip>
          <span
            class="we-steps-label"
            :class="getClasses(child)"
            v-else>{{ child.label }}</span>
        </span>
        <Icon
          :color="getArrowColor(child, index)"
          type="md-arrow-round-forward"
          size="26"
          v-if="index !== renderList.length - 1 && child.isFinish"/>
      </div>
      </Col>
    </Row>
  </div>
</template>
<script>
import _ from 'lodash';
export default {
    props: {
        stepList: {
            type: Array,
            default: () => [],
        },
    },
    data() {
        return {
            // completed 是完成状态，完成状态后不会有其他状态产生；
            // special 是结果集衍生状态，用于判断结果集三个步骤中哪个步骤发生错误；
            // failure 是错误状态，用于标记错误，红色高亮
            // retry 是重试状态，需要在前面追加申请资源失败或执行失败两个状态
            stepsInfo: [
                { label: '任务已提交', value: 'Submitted', status: ['running'], isFinish: false },
                // websocket不返回Inited，所以把第一个接收的execute接口作为Inited
                // 这里是为了防止http方式去请求时，或者长时间未返回状态时，前台去主动请求到Inited状态的情况
                { label: '排队中', value: 'Inited', status: ['running'], isFinish: false },
                { label: '资源申请中', value: 'Scheduled', status: ['running'], isFinish: false },
                { label: '脚本运行中', value: 'Running', status: ['running'], isFinish: false },
                { label: '成功', value: 'Succeed', status: ['running'], isFinish: false },
                { label: '读取结果集中', value: 'ResultLoading', status: ['running'], isFinish: false },
                { label: '完成', value: 'Completed', status: ['completed'], isFinish: false },
                { label: '执行失败', value: 'Failed', status: ['completed', 'failure'], isFinish: false },
                { label: '已取消', value: 'Cancelled', status: ['completed', 'warning'], isFinish: false },
                { label: '获取结果集失败', value: 'FailedToGetResult', status: ['completed', 'failure'], isFinish: true },
                { label: '获取结果集路径失败', value: 'FailedToGetResultPath', status: ['special'], step: 1 },
                { label: '获取结果集列表失败', value: 'FailedToGetResultList', status: ['special'], step: 2 },
                { label: '获取第一个结果集失败', value: 'FailedToGetResultFirst', status: ['special'], step: 3 },
                { label: '等待重试', value: 'WaitForRetry', status: ['retry'], isFinish: false },
                { label: '申请资源失败', value: 'FailedToApply', status: ['failure'], isFinish: true },
                { label: '执行失败', value: 'FailedToExecute', status: ['failure'], isFinish: true },
                { label: '未生成结果集', value: 'notResult', status: ['running', 'warning'], isFinish: false },
            ],
            renderList: [],
            hoverList: [],
        };
    },
    watch: {
        stepList: {
            handler() {
                this.renderSteps();
            },
            deep: true,
        },
    },
    mounted() {
        this.renderSteps();
    },
    methods: {
        renderSteps() {
            this.renderList = [];
            const IS_FINISH = { isFinish: true };
            const IS_LOADING = { isLoading: false };
            const RETRY_STATUS = 'WaitForRetry';
            const FAILED_TO_GET_RESULT_STATUS = 'FailedToGetResult';
            const SCHEDULED_STATUS = 'Scheduled';
            this.stepList.forEach((status, index) => {
                // 要放每个步骤都不是地址引用，否则会出现isFinish这个状态错误
                const step = _.cloneDeep(this.findStep(status));
                // 如果是特殊态的情况，要设置hover的列表，渲染的时候要显示它们的父状态：FailedToGetResult
                if (step.status.indexOf('special') !== -1) {
                    const FailedToGetResult = this.findStep(FAILED_TO_GET_RESULT_STATUS);
                    this.renderList.push(FailedToGetResult);
                    this.setHoverList(step);
                // 如果是重试态，则要在之前插入失败的状态
                } else if (step.value === RETRY_STATUS) {
                    const prevOne = this.stepList[index - 1];
                    const completionValue = prevOne === SCHEDULED_STATUS ? 'FailedToApply' : 'FailedToExecute';
                    const completion = this.findStep(completionValue);
                    this.renderList.push(completion);
                    this.renderList.push(Object.assign(step, IS_FINISH, IS_LOADING));
                // 删除脚本是不会产生结果集的，但会尝试去拿结果，就会出现获取结果集失败→未产生结果集这样的情况，这里的逻辑就是为了避免它
                } else if (step.value === 'notResult') {
                    const lastOne = _.last(this.renderList);
                    const len = this.renderList.length;
                    if (lastOne.value === 'FailedToGetResult') {
                        this.renderList.splice(len - 1, 1);
                    }
                    this.renderList.push(Object.assign(step, IS_FINISH, IS_LOADING));
                } else {
                    this.renderList.push(Object.assign(step, IS_FINISH, IS_LOADING));
                }
                // 最后一个状态
                if (index === this.stepList.length - 1) {
                    // 如果属于运行态，就在最后插入它的下一个状态
                    if (step.status.indexOf('running') !== -1) {
                        step.isLoading = true;
                        const findIndex = this.stepsInfo.findIndex((i) => i.value === status);
                        if (findIndex !== -1) {
                            this.renderList.push(this.stepsInfo[findIndex + 1]);
                        }
                    // 如果是重试态，就在后面插入“资源申请”中这个状态
                    } else if (step.value === RETRY_STATUS) {
                        step.isLoading = true;
                        const scheduled = this.findStep(SCHEDULED_STATUS);
                        this.renderList.push(scheduled);
                    }
                }
            });
        },
        findStep(arg) {
            return this.stepsInfo.find((i) => i.value === arg);
        },
        setHoverList(item) {
            switch (item.step) {
                case 1:
                    this.hoverList = [`1.${item.label}`];
                    break;
                case 2:
                    this.hoverList = ['1.获取结果集路径成功...', `2.${item.label}`];
                    break;
                case 3:
                    this.hoverList = ['1.获取结果集路径成功...', '2.获取结果集列表成功...', `3.${item.label}`];
                    break;
            }
        },
        getArrowColor(child, index) {
            const nextOne = this.renderList[index + 1];
            return this.getIconColor(nextOne);
        },
        getIconColor(child) {
            const COMPLETED_COLOR = '#67c23a';
            const ERROR_COLOR = '#ed4014';
            const CALCELLED_COLOR = '#ddd';
            const UN_FINISHED_COLOR = 'orange';
            if (child.status.indexOf('failure') !== -1) {
                return ERROR_COLOR;
            } else if (child.status.indexOf('warning') !== -1) {
                return CALCELLED_COLOR;
            } else if (child.isFinish && !child.isLoading) {
                return COMPLETED_COLOR;
            }
            return UN_FINISHED_COLOR;
        },
        getIconType(child) {
            const FINISHED = 'md-checkmark-circle';
            const FAILED = 'md-close-circle';
            const WARNING = 'md-alert';
            const LOADING = 'ios-navigate';
            if (child.status.indexOf('failure') !== -1) {
                return FAILED;
            } else if (child.status.indexOf('warning') !== -1) {
                return WARNING;
            } else if (child.isFinish && child.isLoading) {
                return LOADING;
            }
            return FINISHED;
        },
        getClasses(child) {
            if (child.status.indexOf('failure') !== -1) {
                return 'error';
            } else if (child.status.indexOf('warning') !== -1) {
                return 'cancelled';
            } else if (child.isFinish && !child.isLoading) {
                return 'completed';
            } else if (child.isFinish && child.isLoading) {
                return 'loading';
            }
            return '';
        },
    },
};
</script>
