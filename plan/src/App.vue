<template>
    <a-layout id="components-layout-demo-top-side">
        <a-layout-header class="header">
            <div class="logo">
                <!--                <img src="./assets/logo.png"/>-->
                gFOV
            </div>
            <div class="nav">
                <span>Home</span>
                <span>About</span>
                <span>Paper</span>
                <span>Contact</span>
            </div>
        </a-layout-header>
        <a-layout-content style="padding: 0 50px;">
            <a-layout style="padding: 24px; background: #fff; min-height: calc(100vh - 64px - 69px)">
                <h1 class="title">Query Interface</h1>
                <div class=""></div>
                <!--                <a-textarea-->
                <!--                    style="margin-bottom: 24px"-->
                <!--                    placeholder="Please input your query"-->
                <!--                    v-model="query"-->
                <!--                    :auto-size="{ minRows: 5}"/>-->

                <div ref="queryContainer">
                </div>
                <a-button type="primary" icon="search" class="query-btn" @click="performQuery" :loading="loading">
                    Query Now!
                </a-button>
                <a-row>
                    <a-col :span="8">
                        <a-divider v-show="show_result">Original Logical Plan</a-divider>
                        <logical-plan-tree :plan="original_logical_plan" v-show="show_result" :selectable="false"
                                           class="result-plan" ref="oriTree"/>
                    </a-col>

                    <a-col :span="8">
                        <a-divider v-show="show_result">Optimized/Alternative Logical Plan</a-divider>
                        <div style="position: relative">
                            <logical-plan-tree :plan="optimized_logical_plan" v-show="show_result" :selectable="true"
                                               class="result-plan" ref="optTree"/>
                            <!--                            <div v-if="show_result" style="position: absolute; left: 8px; top: 10px">-->
                            <!--                                <a-button type="primary" @click="openCustomPlanPanel">-->
                            <!--                                    Commit a Custom Logical Plan-->
                            <!--                                </a-button>-->
                            <!--                            </div>-->
                            <div v-if="show_result" style="position: absolute; left: 8px; top: 10px">
                                <a-select
                                    ref="select"
                                    v-model="opt_plan_select"
                                    style="width: 160px"
                                    @change="handleChangeAltPlan"
                                >
                                    <a-select-option v-bind:key="altplan['name']" v-for="altplan in alternative_plans"
                                                     :value="altplan['name']">{{ altplan['name'] }}
                                    </a-select-option>
                                </a-select>
                                <div v-if="show_result" style="position: absolute; top: 34px;left: 2px">
                                    <a-button type="primary" :disabled="opt_plan_select === executed_plan" :loading="loading" @click="commitAltPlan">
                                        Commit an Alternative Plan
                                    </a-button>
                                </div>
                            </div>
                            <div v-if="show_result && alternative_plans.length >= 1"
                                 style="position: absolute; right: 8px; top: 10px;
                                 padding: 10px; border-radius: 5px;
                                 background: rgba(0, 0, 0, 0.1); font-size:16px;">
                                Δcost: {{ alternative_plans.filter(x => x['name'] === opt_plan_select)[0]['cost'] }}
                            </div>
                        </div>
                    </a-col>

                    <a-col :span="8">
                        <a-divider v-show="show_result">Physical Plan</a-divider>
                        <div style="position:relative;">
                            <plan-tree :plan="BGP_plan" v-show="show_result" :enable_tip="true" class="result-plan"
                                       :msg="opt_plan_select !== executed_plan ? 'The physical plan (BGP) can be obtained only after the logical plan is executed.' : null "
                                       ref="bgpTree"/>
                            <!--                            <div v-if="show_result" style="position: absolute; top: 10px;left: 8px">-->
                            <!--                                <a-button type="primary" @click="openCustomPlanPanel">-->
                            <!--                                    Commit a Custom BGP Plan-->
                            <!--                                </a-button>-->
                            <!--                            </div>-->
                        </div>
                    </a-col>

                </a-row>
                <a-divider v-if="show_result">Execution time: {{ result.QueryTime }} ms</a-divider>
                <a-row>
                    <a-col :span="24">
                        <a-table v-show="show_result" :columns="getCol" :data-source="getData" class="result-table">
                            <a slot="name" slot-scope="text">{{ text }}</a>
                        </a-table>
                    </a-col>
                </a-row>
                <!--                <a-row>-->
                <!--                    -->
                <!--                </a-row>-->


            </a-layout>
        </a-layout-content>
        <a-layout-footer style="text-align: center">
            ©2023 Created by pkumod
        </a-layout-footer>
    </a-layout>
</template>

<script>
import axios from 'axios'
// import JSONResult from './components/result.json'
import PlanTree from './components/planTree'
import LogicalPlanTree from './components/logicalPlanTree'
import Yasqe from '@triply/yasqe'


const throttle = (fn, delay = 2000) => {
    let lastTime = 0, timer = null

    return function () {
        let _this = this
        let _arguments = arguments
        let now = new Date().getTime()
        clearTimeout(timer)
        // 判断上次触发的时间和本次触发的时间差是否小于delay,创建一个timer
        if (now - lastTime < delay) {
            timer = setTimeout(function () {
                lastTime = now
                console.log('执行器触发')
                fn.apply(_this, _arguments)
            }, delay)
        } else {
            // 否则可以直接执行
            lastTime = now
            console.log('直接触发')
            fn.apply(_this, _arguments)
        }
    }
}

export default {
    components: {PlanTree, LogicalPlanTree},
    data() {
        return {
            query: '',
            loading: false,
            show_result: false,
            result: {},
            yasqe: null,
            original_logical_plan: '',
            optimized_logical_plan: '',
            alternative_plans: [],
            opt_plan_select: 'Optimized Plan',
            executed_plan: 'Optimized Plan',
            plan: '',
            BGP_plan: '',
            new_plan: '',
            new_plan_loading: false,
            show_custom_plan_panel: false,
            new_plan_query_loading: false,
            // query_url: 'http://localhost:5000/query',
            query_url: '/query'
        }
    },
    computed: {
        getCol() {
            if (!('head' in this.result))
                return []
            return this.result.head.vars.map(e => {
                return {
                    title: e,
                    dataIndex: e + '.value',
                    key: e,
                }
            })
        },
        getData() {
            if (!('results' in this.result))
                return []
            return this.result.results.bindings
        },
    },
    mounted() {
        this.initYasqe()
        this.$root.$on('renderBGPPlan', (BGP_id) => {
            // 只显示optimized plan的BGP
            if(this.opt_plan_select === this.executed_plan)
                this.renderBGP(BGP_id)
        })
    },
    methods: {
        performQuery() {
            this.loading = true
            axios.post(this.query_url, {
                query: this.yasqe.getValue(),
                plan: '',
            }).then(res => {
                this.result = res.data
                this.plan = res.data.Plan
                this.original_logical_plan = res.data.OriginalLogicalPlan
                this.optimized_logical_plan = res.data.OptimizedLogicalPlan
                this.alternative_plans = [{
                    name: 'Original Plan',
                    plan: this.original_logical_plan,
                    cost: 0
                }, {
                    name: 'Optimized Plan',
                    plan: this.optimized_logical_plan,
                    cost: res.data.OptimizedLogicalPlanCost
                }]
                this.executed_plan = 'Optimized Plan'
                try {
                    // {"(GGP((b1())(UN(GGP((b2()))(GGP((b3())))))(OPT(GGP((b1,4()))))))":-131}
                    let altplans = JSON.parse(res.data.AltPlanCost)
                    for (let plan in altplans) {
                        this.alternative_plans.push({
                            name: `Alternative Plan ${this.alternative_plans.length - 1}`,
                            plan: plan,
                            cost: altplans[plan]
                        })
                    }
                } catch (e) {
                    console.log(e)
                }

                this.loading = false
                this.show_result = true
                this.$nextTick(() => this.$refs.oriTree.$emit('renderPlan'))
                this.$nextTick(() => this.$refs.optTree.$emit('renderPlan'))
            })
        },
        handleChangeAltPlan(v) {
            this.opt_plan_select = v
            if (this.opt_plan_select !== 'Optimized Plan') {
                this.$refs.bgpTree.$emit('clearPlan')
            }
            this.optimized_logical_plan = this.alternative_plans.filter(e => e.name === v)[0].plan
            this.$nextTick(() => this.$refs.optTree.$emit('renderPlan'))
        },
        renderBGP(BGP_id) {
            if (this.plan === '')
                console.error('No plan')
            let BGP_plans = this.plan.split(');')
            this.BGP_plan = BGP_plans.filter(e => e.startsWith(BGP_id))[0].split('(')[1]
            console.log(this.BGP_plan)
            this.$nextTick(() => this.$refs.bgpTree.$emit('renderPlan'))
        },
        initYasqe() {
            console.log(this.$refs.queryContainer)
            this.yasqe = new Yasqe(this.$refs.queryContainer,
                {
                    createShareableLink: false,
                    showQueryButton: false,
                    value: this.query,
                })
        },
        openCustomPlanPanel() {
            this.new_plan = this.plan.split(';').slice(0, 2).join(';')
            this.show_custom_plan_panel = true
            this.$nextTick(() => {
                this.$refs.oldtree.$emit('renderPlan')
                this.$refs.newtree.$emit('renderPlan')
            })
        },
        throttleInput: throttle(function (...args) {
            this.checkNewPlan(...args)
        }, 2000),
        checkNewPlan() {
            this.new_plan_loading = false
            this.$refs.newtree.$emit('renderPlan')
        },
        query_with_plan() {
            console.log('query:', this.yasqe.getValue())
            console.log('plan:', this.plan)
            console.log('new_plan:', this.new_plan)
            this.new_plan_query_loading = true
            axios.post(this.query_url, {
                query: this.yasqe.getValue(),
                plan: this.new_plan,
            }).then(res => {
                console.log(res)
                this.result = res.data
                this.plan = res.data.Plan
                this.new_plan_query_loading = false
                this.show_custom_plan_panel = false
                this.show_result = true
                this.$nextTick(() => this.$refs.tree.$emit('renderPlan'))
            })
        },
        commitAltPlan() {
            this.new_plan = this.alternative_plans.filter(e => e.name === this.opt_plan_select)[0].plan
            this.executed_plan = this.opt_plan_select
            this.loading = true
            axios.post(this.query_url + '_opt', {
                query: this.yasqe.getValue(),
                plan: this.new_plan,
            }).then(res => {
                this.result = res.data
                this.plan = res.data.Plan
                this.loading = false
                this.show_result = true
                this.$nextTick(() => this.$refs.oriTree.$emit('renderPlan'))
            })
        }
    },

}
</script>

<style scoped>
@import '~@triply/yasqe/build/yasqe.min.css';
</style>
<style>
.CodeMirror {
    z-index: 0 !important;
}
</style>
<style>
.logo {
    color: #fff;
    font-size: 1.5em;
    font-weight: bold;
    font-style: italic;
    max-width: 300px;
    display: inline-block;
}

.logo img {
    width: auto;
    height: 57px;
    float: left;
}

.title {
    text-align: center;
    font-size: 30px;
    font-weight: bold;
}

.nav {
    float: right;
    color: #aaa;
    font-size: 1.2em;
}

.nav span {
    display: inline-block;
    margin-right: 20px;
    cursor: pointer;
}

.nav span:hover {
    color: #fff;
}

.result-json {
    overflow-x: scroll;
}

.result-table {
    overflow-x: auto;
}

.result-plan {
    border: 1px solid #eee;
    margin: 5px;
    border-radius: 8px;
}
</style>
