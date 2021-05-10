<template>
    <div tabindex="-1" class="mail-wrapper" @keyup="handleWrapperKeyup" @click="handleWrapperClick">
        <template v-for="(item, index) in arr">
            <div v-clickoutside="handleClickOutside.bind(this, item)" v-if="item.type === 'block'" :key="item.value" class="mail-block" :class="{selected: item.selected, error: item.error}" @click.stop="handleClick(item)" @dblclick="blockDbClick(index)">{{item.value}}</div>
            <div :key="item.type + id" v-else class="mail-text"> <!-- 输入框的key为input 这样可以复用输入框dom,同一时间该组件内只会存在一个input输入框 -->
                <input ref="mailInput" v-model="inputValue" type="text" @keyup="handleKeyup" @blur="handleBlur">
                <span>{{inputValue}}WW</span>
            </div>
        </template>
    </div>
</template>

<script>
import Clickoutside from './clickoutside';
    export default {
        directives: { Clickoutside },
        props: {
            separator: {
                type: String,
                default: ';',
            },
            value: {
                type: Array,
                default () {
                    return [{type:'block', value:'2609200106@qq.com'}, {type:'block', value: '世纪东方垃圾分类的考试缴费'}];
                }
            },
            rule: {
                type: Function,
                default: () => {
                    return true;
                }
            }
        },
        data() {
            return {
                id: this._uid,
                arr: this.value.map(el => {
                        el.selected = false;
                        el.error = false;
                        return el;
                    }).concat({type:'input'}),
                inputValue:'',
            }
        },
        mounted () {
            this.arr.forEach(el => {
                if (this.rule(el.value)) {
                    el.error = false;
                } else {
                    el.error = true;
                }
            })
            this.$forceUpdate();
        },
        methods: {
            handleWrapperClick () {
                this.focusInput();
            },
            handleWrapperKeyup (e) {
                console.log(e, 'wrapper')
                if (e.key === 'Backspace') {
                    let selectedIndex = this.arr.findIndex(el => el.selected);
                    selectedIndex !== -1 && this.arr.splice(selectedIndex, 1);
                }
            },
            handleClick (item) {
                this.arr.forEach(el => el.selected = false);
                item.selected = true;
                this.$forceUpdate();
            },
            handleClickOutside (item) {
                item.selected = false;
                this.$forceUpdate();
                // this.arr.forEach(el => this.$set(el, 'selected', false));
            },
            swapItem(arr, fromIndex, toIndex) { // 交换两个元素位置,eg. arr = [1,2,3,4,5]，将3与1交换swapItem(arr, 2, 0)，结果是[3,2,1,4,5]
                arr[toIndex] = arr.splice(fromIndex, 1, arr[toIndex])[0];
                return arr;
            },
            handleBlur (e) {
                // console.log(e.target.value)
                if (!e.target.value) {
                    // 如果输入框没有内容，失焦后需要把输入框移动最后
                    let index = this.arr.findIndex(el => el.type === 'input');
                    if (index !== this.arr.length - 1) { // 如果输入框已经在末尾了，就不需要再去换位置了
                        let item = this.arr.splice(index, 1)[0];
                        this.arr.push(item);
                    }
                    return
                }
                // 如果输入框是有内容的，则生成新的块
                let index = this.arr.findIndex(el => el.type === 'input');
                let isError = this.rule(e.target.value) ? false : true;
                this.arr.splice(index, 1, {type: 'block', value: e.target.value, selected: false, error: isError})
                this.arr.push({type:'input'})
                this.inputValue = '';

            },
            blockDbClick (index) {

                // 将双击项的值赋值给input输入框
                let value = this.arr[index].value;
                this.inputValue = value;

                // 将输入框移到 双击项的位置
                let inputIndex = this.arr.findIndex(el => el.type === 'input');
                this.arr.splice(index, 1, this.arr[inputIndex]);
                this.arr.splice(inputIndex, 1);
                
                // 让输入框聚焦
                this.$nextTick(() => { // 需要放在nextTick里去focus，不然focus了，vue更新后又失焦了
                    this.$refs.mailInput[0].focus();
                    this.$refs.mailInput[0].selectionStart = 0;
                    this.$refs.mailInput[0].selectionEnd = this.inputValue.length;
                })
            },
            focusInput () {
                this.$nextTick(() => { // 需要放在nextTick里去focus，不然focus了，vue更新后又失焦了
                    this.$refs.mailInput[0].focus();
                });
            },
            handleKeyup (e) {
                
                if ((e.key === this.separator || e.key === ' ') && e.target.selectionEnd === this.inputValue.length) { // 输入的分隔符必须是在光标末尾，中间输入的分隔符不算，比如 ‘haha;ff’是不算的，‘hahaff;’才算
                    // 在input输入框前插入一个 新的block
                    let inputIndex = this.arr.findIndex(el => el.type === 'input');
                    let newValue = this.inputValue.slice(0, this.inputValue.length - 1);
                    let isError = this.rule(newValue) ? false : true;
                    this.arr.splice(inputIndex, 0, {type: 'block', value: newValue, selected: false, error: isError});
                    this.focusInput();
                    this.inputValue = '';
                }
                if (e.key === 'Backspace') {  // 退格键
                   if (this.inputValue.length === 0) {
                       let inputIndex = this.arr.findIndex(el => el.type === 'input');
                       if (inputIndex > 0) {
                           this.arr[inputIndex - 1].selected = true;
                            this.$nextTick(() => { // 需要放在nextTick里去focus，不然focus了，vue更新后又失焦了
                                this.$refs.mailInput[0].blur();
                            }); 
                            this.$forceUpdate();
                       }
                   } 
                }
                if (e.key === 'ArrowLeft') {
                    if (!this.inputValue) { //输入框的值为空 并且输入框不在第一项
                        let index = this.arr.findIndex(el => el.type === 'input');
                        if (index !== 0) {
                            this.swapItem(this.arr, index, index - 1);
                            this.focusInput();
                        }
                        
                    }
                }

                if (e.key === 'ArrowRight') {
                    if (!this.inputValue) { //输入框的值为空 并且输入框不在最后一项
                        console.log(this.arr[0].type);
                        let idx = this.arr.findIndex(item => item.type === 'input');
                        if (idx !== this.arr.length - 1) {
                            this.swapItem(this.arr, idx, idx + 1);
                            this.focusInput();
                        }
                         
                    }
                }
                console.log(e)
            }
        }
    }
</script>

<style lang="scss" scoped>
.mail-wrapper {
    cursor: text;
    border: 1px solid rgba(224, 227, 230, 1);
    background: #fff;
    overflow: hidden;
    padding: 4px;
    .mail-block {
        cursor: pointer;
        &.error {
            color: #FD6464;
            background-color: transparent;
        }
        &.selected {
            background-color: rgba(64, 132, 255, 0.2);
        }

        color: #262B39;
        float: left;
        height: 24px;
        line-height: 24px;
        background-color: rgba(118, 119, 124, 0.1);
        margin-right: 4px;
        padding: 0 4px;
    }
    .mail-text {
        cursor: pointer;
        height: 24px;
        float: left;
        position: relative;
        margin-right: -23px;
        input {
            &::selection {
                background-color: #fde9be;
                color: #262B39;
            } 
            color: #262B39;
            height: 24px;
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            border: none;
            background: transparent;
        }
        span {
            visibility: hidden;
        }

    }
}
</style>