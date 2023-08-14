<template>
    <div>
        <div class="mask" v-if="msg">
            <div class="msg">{{ msg }}</div>
        </div>
        <div class="plan-container" ref="container">
        </div>
    </div>
</template>

<script>
import cytoscape from 'cytoscape'
import popper from 'cytoscape-popper'
import dagre from 'cytoscape-dagre'

cytoscape.use(popper)
cytoscape.use(dagre)

export default {
    name: 'LogicalPlanTree',
    props: ['plan', 'msg', 'selectable'],
    data() {
        return {
            tips: {},
            showDetailPlan: true,
        }
    },
    mounted() {
        this.initCy()
        this.$on('renderPlan', () => this.renderPlan())
    },
    methods: {
        switchDetailTip(checked) {
            this.showDetailPlan = checked
            if (checked)
                for (let tip in this.tips)
                    this.tips[tip].show()
            else
                for (let tip in this.tips)
                    this.tips[tip].hide()
        },
        initCy() {
            this.cy = cytoscape({
                container: this.$refs['container'],
                // userZoomingEnabled: false,
                // userPanningEnabled: false,
                elements: [],
                style: [
                    {
                        selector: 'node',
                        style: {
                            'background-color': '#FF8C05',
                            'label': 'data(label)',
                        },
                    },
                    {
                        selector: 'edge',
                        style: {
                            'width': 3,
                            'line-color': '#ccc',
                            'target-arrow-color': '#ccc',
                            'target-arrow-shape': 'triangle',
                        },
                    },
                    {
                        selector: '.OPT',
                        style: {
                            'shape': 'round-rectangle',
                            'background-color': '#4499EE',
                            'width': 35,
                            'height': 35,
                            'text-valign': 'bottom',
                            'text-halign': 'center',
                            'text-margin-y': -22,
                            'color': '#fff',
                            'font-size': 10,
                        },
                    },
                    {
                        selector: '.GGP',
                        style: {
                            'shape': 'round-rectangle',
                            'background-color': '#705b8b',
                            'width': 35,
                            'height': 35,
                            'text-valign': 'bottom',
                            'text-halign': 'center',
                            'text-margin-y': -22,
                            'color': '#fff',
                            'font-size': 10,
                        },
                    },
                    {
                        selector: '.UN',
                        style: {
                            'shape': 'round-rectangle',
                            'background-color': '#439f01',
                            'width': 35,
                            'height': 35,
                            'text-valign': 'bottom',
                            'text-halign': 'center',
                            'text-margin-y': -22,
                            'color': '#fff',
                            'font-size': 10,
                        },
                    },
                    {
                        selector: '.B:selected',
                        style: {
                            'border-width': 2,
                            'border-color': '#23d802',
                        },
                    },
                ],
                layout: {
                    name: 'breadthfirst',
                    directed: true,
                    padding: 10,
                },
            })
        },
        renderPlan() {
            let plan = this.plan
            let cy = this.cy
            cy.elements().remove()
            cy.reset()

            //(GGP
            //  (b1
            //      (?stu <rdf:type> <ub:UndergraduateStudent>.?prof <ub:researchInterest> <Research1>.?course <rdf:type> <ub:Course>.?prof <ub:teacherOf> ?course.?stu <ub:advisor> ?prof.)
            //  )
            //  (UN
            //      (GGP
            //          (b2
            //              (?stu <ub:takesCourse> ?course.)))(GGP(b3(?stu <ub:teachingAssistantOf> ?course.)
            //          )
            //       )
            //   )
            //   (OPT
            //      (GGP
            //          (b4
            //              (?pub <ub:publicationAuthor> ?prof.?prof <ub:doctoralDegreeFrom> ?uni.?pub <rdf:type> <ub:Publication>.)
            //          )
            //       )
            //   )
            //)
            let that = this

            function parseLogicalTree(plan, cy) {
                plan = plan.replace(/\s/g, '').split(/(\()|(\))/).filter(e => e !== undefined && e !== '')
                if (plan.length <= 2)
                    console.error('Invalid plan', plan)
                plan = plan.slice(1, -1)
                // First element is the root node
                let root = plan[0] + '---' + getUid()
                addNode(root, {label: plan[0], classes: plan[0]}, cy)

                // traverse the plan and construct a tree
                let stack = []
                let now_node = root
                for (let node of plan.slice(1)) {
                    if (node === '(') {
                        stack.push(now_node)
                    } else if (node === ')') {
                        now_node = stack.pop()
                    } else {
                        if (node.startsWith('?')) continue
                        let classes = node
                        if (classes.startsWith('b')) {
                            classes = 'B'
                            if (!that.selectable) classes += 'J'
                        }
                        let label = node + '---' + getUid()
                        addNode(label, {label: node, classes}, cy)
                        addEdge(now_node, label, cy)
                        now_node = label
                    }
                }
            }

            parseLogicalTree(plan, cy)
            // Register Event
            if (this.selectable)
                cy.$('.B').on('tap', evt => {
                    this.$root.$emit('renderBGPPlan', evt.target.id().split('---')[0])
                })
            cy.layout({
                name: 'dagre',
                siblingDistance: 80,
            }).run()
            cy.elements().lock()
            setTimeout(() => {
                cy.animate({fit: {eles: cy.elements(), padding: 20}})
            }, 500)
        },
    },
}

function addNode(id, options, cy) {
    cy.add({
        group: 'nodes',
        data: {
            id: id,
            label: options['label'],
        },
        classes: options['classes'],
    })
}

function addEdge(source, target, cy) {
    cy.add({
        group: 'edges',
        data: {
            source: source,
            target: target,
        },
    })
}

function getUid() {
    let s = []
    let hexDigits = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ'
    for (let i = 0; i < 12; i++) {
        s[i] = hexDigits.substr(Math.floor(Math.random() * 0x10), 1)
    }
    return s.join('')
}


</script>

<style>
.mask {
    position: absolute;
    height: 100%;
    width: 100%;
    z-index: 200000;
    background: rgba(241, 241, 241, .4);
}

.msg {
    position: absolute;
    top: 50%;
    height: 55px;
    padding: 10px;
    line-height: 1;
    background: #fff;
    font-size: 25px;
    text-align: center;
    width: 100%;
}

.plan-container {
    height: 700px;
    width: 100%;
    position: relative;
    overflow: hidden;
}

</style>
