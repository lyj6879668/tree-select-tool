<template>
    <div>
        <Input
            v-if="isFilterable"
            v-model="searchValue"
            placeholder="查询..."
            search
            @click.native="searchTree"
            @on-change="handleSearchChange"
        />
        <Tree
            ref="tree"
            :data="data"
            :show-checkbox="isShowCheckbox"
            @on-select-change="handleSelect"
            @on-check-change="handleCheck"
        ></Tree>
    </div>
</template>

<script>
import Emitter from '@/libs/emitter'
export default {
    name: 'SelectTreeTool',
    mixins: [Emitter],
    inject: ['parent'],
    props: {
        isShowCheckbox: {
            type: Boolean
        },
        loadData: {
            type: Array,
            default: () => []
        },
        loadDataUrl: String,
        checkedValue: {
            type: Array
        },
        defaultId: {
            type: String
        },
        defaultTitle: {
            type: String
        },
        isFilterable: {
            type: Boolean,
            default: false
        }
    },
    data() {
        return {
            data: [],
            checkedList: [],
            topNodes: [],
            searchValue: '',
        }
    },
    methods: {
        checkEmit(value, label) {
            this.dispatch('iSelect', 'on-select-selected', {
                value,
                label
            })
            this.$emit('on-select-selected', {
                value,
                label
            })
        },
        selectEmit(value, label) {
            this.parent.$emit('on-select-selected'.concat({
                value,
                label
            }))
            this.$emit('on-select-selected', {
                value,
                label
            })
        },
        checkDataByParent() {
            if (this.defaultId != 'id' || this.defaultTitle != 'title') {
                this.checkDefaultData(this.data, this.defaultId, this.defaultTitle)
            }
            if (this.checkedValue && this.checkedValue.length) {
                this.checkData(this.data, this.checkedValue, true)
            }
        },
        checkDefaultData(data, id, title) {
            let _this = this
            data.forEach(item => {
                if (item[id]) {
                    _this.$set(item, 'id', item[id])
                }
                if (item[title]) {
                    _this.$set(item, 'title', item[title])
                }
                if (item.children && item.children.length) {
                    _this.checkDefaultData(item.children, id, title)
                }
            })
        },
        checkData(data, checkedList, isDefault) {
            let _this = this
            data.forEach(item => {
                if (checkedList.includes(item.id)) {
                    _this.$set(item, _this.isShowCheckbox ? 'checked' : 'selected', true)
                    _this.topNodes.push(item)
                    if (isDefault) {
                        _this.$nextTick(() => {
                            _this.checkEmit(item.id, item.title)
                        })
                    }
                }
                else {
                    _this.$set(item, _this.isShowCheckbox ? 'checked' : 'selected', false)
                }
                if (item.children && item.children.length) {
                    _this.$set(item, 'expand', true)
                    _this.checkData(item.children, checkedList, isDefault)
                }
            })
        },
        handleSelect(data) {
            this.$emit('on-select', data)
            this.parent.$emit('on-change', data)
            this.parent.handleClear()
            data.forEach(item => {
                this.checkEmit(item.id, item.title)
            })
        },
        handleCheck(data) {
            let deleteNodes = [];
            data.forEach(item => {
                if (item.children) {
                    deleteNodes = [...deleteNodes, ...item.children]
                }
            })
            let deleteIds = deleteNodes.map(item => { return item.id })
            this.topNodes = data.filter(item => {
                return deleteIds.indexOf(item.id) == -1
            })
            this.$emit('on-check', this.topNodes)
            this.parent.$emit('on-change', this.topNodes)
            this.parent.handleClear()
            this.topNodes.forEach(item => {
                this.checkEmit(item.id, item.title)
            })
        },
        changeTreeBySelect(isChangeByTree, deleteItem) {
            if (!isChangeByTree) {
                let checkedList = this.topNodes.map(item => item.id).filter(item => { return item != deleteItem.value })
                this.topNodes.splice(0)
                this.checkData(this.data, checkedList, false)
            }
        },
        searchTree() {
            if (this.isFilterable) {
                this.changeTreeStyle(this.data, this.searchValue)
            }
        },
        handleSearchChange(input) {
            this.searchTree()
        },
        changeTreeStyle(data, value) {
            let _this = this
            data.forEach(item => {
                if (value) {
                    if (item.title.includes(value)) {
                        _this.$set(item, 'render', (h, { root, node, data }) => {
                            return h('span', {
                                style: {
                                    color: '#2d8cf0'
                                }
                            }, data.title)
                        })
                    }
                }
                else {
                    _this.$set(item, 'render', null)
                }
                if (item.children && item.children.length) {
                    _this.changeTreeStyle(item.children, value)
                }
            })
        }
    },
    mounted() {
        if (this.loadData && this.loadData.length) {
            this.data = JSON.parse(JSON.stringify(this.loadData))
            this.checkDataByParent()
        }
        else {
            this.axios.get({ url: this.loadDataUrl }, {}, res => {
                this.data = JSON.parse(JSON.stringify(res.data))
                this.checkDataByParent()
            })
        }
    }
}
</script>
