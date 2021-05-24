<template>
    <div class="py-input-wrapper">
        <div
            ref="multiInput"
            tabindex="-1"
            class="py-input-content"
            @keyup="handleWrapperKeyup"
            @click="handleWrapperClick"
            :class="{focus:focused, error: !!errorInfo}"
            :style="{width: width, height: height}"
        >
            <span
                ref="placeHolder"
                v-show="isEmpty"
                class="placeholder"
            >{{ placeholder }}</span>
            <template v-for="(item, index) in arr">
                <div
                    :ref="'block' + item.id"
                    v-clickoutside="handleClickOutside.bind(this, item)"
                    v-if="item.type === 'block'"
                    :key="item.id"
                    class="mail-block"
                    :class="{selected: item.selected, error: item.error}"
                    @click.stop="handleClick(item)"
                    @dblclick="blockDbClick(index)"
                >{{ item.value }}</div>
                <div
                    :key="item.type + id"
                    v-else
                    class="mail-text"
                > <!-- 输入框的key为input 这样可以复用输入框dom,同一时间该组件内只会存在一个input输入框 -->
                    <input
                        ref="mailInput"
                        v-model="inputValue"
                        type="text"
                        @keyup="handleKeyup"
                        @blur="handleBlur"
                    >
                    <span>{{ inputValue }}WW</span>
                </div>
            </template>
        </div>
        <div
            v-if="errorInfo"
            class="error-tip"
        >
            {{ errorInfo }}
        </div>
    </div>

</template>

<script>
import Clickoutside from 'iview/src/directives/clickoutside';
let blockId = 0;
export default {
    directives: { Clickoutside },
    props: {
        placeholder: {
            type: String,
            default: '',
        },
        width: {
            type: String,
            default: '400px',
        },
        height: {
            type: String,
            default: '64px',
        },
        separator: {
            type: String,
            default: ';',
        },
        value: {
            type: Array,
            default () {
                return [];
            },
        },
        rule: {
            type: Array,
            default () {
                return [];
            },
        },
    },
    data () {
        return {
            errorInfo: '',
            focused: false,
            id: this._uid,
            arr: this.value.map(el => {
                let obj = {
                    id: ++blockId,
                    value: el,
                    selected: false,
                    error: false,
                    type: 'block',
                };
                return obj;
            }).concat({ type: 'input' }),
            inputValue: '',
        };
    },
    computed: {
        isEmpty () {
            return this.arr.length === 1 && this.inputValue.length === 0 ;
        },
    },
    watch: {
        value: {
            deep: true,
            handler () {
                this.arr = this.value.map(el => {
                    let isError = !!this.validateBlock(el);
                    let obj = {
                        id: ++blockId,
                        value: el,
                        selected: false,
                        error: isError,
                        type: 'block',
                    };
                    return obj;
                }).concat({ type: 'input' });
            },
        },

        arr: {
            deep: true,
            handler () {
                this.validateAllBlocks();
            },
        },
        /*
         * arr: {
         *     deep: true,
         *     handler (val, oldVal) {
         *         // let blocks = val.filter(el => el.type === 'block');
         *         // let oldBlocks = oldVal.filter(el => el.type === 'block');
         *         // let i = blocks.length, j = oldBlocks.length;
         */


        /*
         *         // this.$emit('input', this.arr);
         *     }
         * }
         */
    },
    mounted () {
        this.validateAllBlocks();
        this.$forceUpdate();
    },
    methods: {
        validateBlock (val) {
            let unMatchRule = this.rule.find(r => !r.validator(val));
            if (unMatchRule) {
                return unMatchRule.message;
            } else {
                return null;
            }
        },
        validateAllBlocks () {
            let hasError = false;
            this.arr.filter(e => e.type === 'block').forEach(el => {
                let errorMessage = this.validateBlock(el.value);
                if (!errorMessage) {
                    el.error = false;
                } else {
                    el.error = true;
                    hasError = true;
                    this.errorInfo = errorMessage;
                }
            });
            if (!hasError) {
                this.errorInfo = '';
            }
            return this.errorInfo;
        },
        getNormalValue (arr) {
            let value = arr.filter(el => el.type === 'block').map(item => item.value);
            return value;
        },
        handleWrapperClick (e) {
            if (e.target !== this.$refs.multiInput && e.target !== this.$refs.placeHolder) {
                return;
            }


            let inputIndex = this.arr.findIndex(el => el.type === 'input');
            let inputItem = this.arr.splice(inputIndex, 1)[0];
            this.inputValue = '';
            let insertIndex = this.arr.length; // 如果没有在下面那个循环内对insertIndex赋值，说明没有找到合适的插入位置，那么默认就往末尾插入
            for (let i = 0, len = this.arr.length; i < len; i ++) {
                let item = this.arr[i];
                if (item.type === 'block') {
                    let { left, bottom } = this.$refs[`block${item.id}`][0].getBoundingClientRect();
                    if (e.clientX < left && e.clientY < bottom) {
                        insertIndex = i;
                        break;
                    }
                }
            }

            this.arr.splice(insertIndex, 0, inputItem);
            this.focusInput();
        },
        handleWrapperKeyup (e) {
            if (e.key === 'Backspace') {
                let selectedIndex = this.arr.findIndex(el => el.selected);
                selectedIndex !== -1 && this.arr.splice(selectedIndex, 1);
                this.$emit('on-change', this.getNormalValue(this.arr));
                this.$emit('input', this.getNormalValue(this.arr));
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
        swapItem (arr, fromIndex, toIndex) { // 交换两个元素位置,eg. arr = [1,2,3,4,5]，将3与1交换swapItem(arr, 2, 0)，结果是[3,2,1,4,5]
            arr[toIndex] = arr.splice(fromIndex, 1, arr[toIndex])[0];
            return arr;
        },
        handleBlur (e) {
            // console.log(e.target.value)
            this.focused = false;
            if (!e.target.value) {
                /*
                 * 如果输入框没有内容，失焦后需要把输入框移动最后
                 * let index = this.arr.findIndex(el => el.type === 'input');
                 * if (index !== this.arr.length - 1) { // 如果输入框已经在末尾了，就不需要再去换位置了
                 *     let item = this.arr.splice(index, 1)[0];
                 *     this.arr.push(item);
                 * }
                 */
                return;
            }
            // 如果输入框是有内容的，则生成新的块
            let index = this.arr.findIndex(el => el.type === 'input');
            let isError = !!this.validateBlock(e.target.value);
            this.arr.splice(index, 1, { type: 'block', value: e.target.value, selected: false, error: isError, id: ++blockId });
            this.arr.push({ type: 'input' });
            this.inputValue = '';
            this.$emit('on-change', this.getNormalValue(this.arr));
            this.$emit('input', this.getNormalValue(this.arr));
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
            });
        },
        focusInput () {
            this.$nextTick(() => { // 需要放在nextTick里去focus，不然focus了，vue更新后又失焦了
                this.focused = true;
                this.$refs.mailInput[0].focus();
            });
        },
        handleKeyup (e) {
            if ((e.key === this.separator || e.key === ' ') && e.target.selectionEnd === this.inputValue.length) { // 输入的分隔符必须是在光标末尾，中间输入的分隔符不算，比如 ‘haha;ff’是不算的，‘hahaff;’才算
                // 在input输入框前插入一个 新的block,生成新的块
                let inputIndex = this.arr.findIndex(el => el.type === 'input');
                let newValue = this.inputValue.slice(0, this.inputValue.length - 1);
                let isError = !!this.validateBlock(e.target.value);
                this.arr.splice(inputIndex, 0, { type: 'block', value: newValue, selected: false, error: isError, id: ++blockId });
                this.focusInput();
                this.inputValue = '';
                this.$emit('on-change', this.getNormalValue(this.arr));
                this.$emit('input', this.getNormalValue(this.arr));
            }
            if (e.key === 'Backspace') { // 退格键
                if (this.inputValue.length === 0) {
                    let inputIndex = this.arr.findIndex(el => el.type === 'input');
                    if (inputIndex > 0) {
                        this.arr[inputIndex - 1].selected = true;
                        this.$nextTick(() => { // 需要放在nextTick里去focus，不然focus了，vue更新后又失焦了
                            this.$refs.mailInput[0].blur();
                            this.$refs.multiInput.focus();
                        });
                        this.$forceUpdate();
                    }
                }
                e.stopPropagation(); // 不让事件冒泡，否则multiInput div也会监听到，然后把邮件块删掉一个。
            }
            if (e.key === 'ArrowLeft') {
                if (!this.inputValue) { // 输入框的值为空 并且输入框不在第一项
                    let index = this.arr.findIndex(el => el.type === 'input');
                    if (index !== 0) {
                        this.swapItem(this.arr, index, index - 1);
                        this.focusInput();
                    }
                }
            }

            if (e.key === 'ArrowRight') {
                if (!this.inputValue) { // 输入框的值为空 并且输入框不在最后一项
                    let idx = this.arr.findIndex(item => item.type === 'input');
                    if (idx !== this.arr.length - 1) {
                        this.swapItem(this.arr, idx, idx + 1);
                        this.focusInput();
                    }
                }
            }
        },
    },
};
</script>

<style lang="scss" scoped>
.py-input-wrapper {
    position: relative;

    .py-input-content {
        cursor: text;
        border-radius: 4px;
        border: 1px solid rgba(224, 227, 230, 1);
        background: #fff;
        overflow: hidden;
        padding: 2px 6px 6px;
        position: relative;
        transition: border .2s ease-in-out, box-shadow .2s ease-in-out;

        &.focus {
            box-shadow: 0 0 0 2px rgb(68 126 231 / 20%);
            border-color: #6998ec;

            &.error {
                box-shadow: 0 0 0 2px rgb(237 64 20 / 20%);
            }
        }

        &:hover {
            border-color: #6998ec;
        }

        &.error {
            border-color: #ed4014;
        }

        .placeholder {
            color: #b9bac1;
            position: absolute;
            left: 8px;
            top: 8px;
        }

        .mail-block {
            cursor: pointer;

            &.error {
                color: #fd6464;
                background-color: transparent;
            }

            &.selected {
                background-color: rgba(64, 132, 255, .2);
            }

            color: #262b39;
            float: left;
            height: 24px;
            line-height: 24px;
            background-color: rgba(118, 119, 124, .1);
            margin-right: 4px;
            margin-top: 4px;
            padding: 0 4px;
        }

        .mail-text {
            cursor: pointer;
            height: 24px;
            float: left;
            position: relative;
            margin-right: -23px;
            margin-top: 4px;

            input {
                &::selection {
                    background-color: #fde9be;
                    color: #262b39;
                }

                color: #262b39;
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

    .error-tip {
        position: absolute;
        color: #ed4014;
        bottom: -21px;
        left: 4px;
    }
}

</style>
