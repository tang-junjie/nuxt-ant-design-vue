<template>
    <a-layout id="components-layout-mobile-top"
              class="layout"
              :class="(drawer ? 'drawer-containel': '')">
        <fs-head :SelectedKey="key" :SelectedType="type" :centerKey="centerKey" @openDrawer="openDrawer" @closeDrawer="closeDrawer"/>
        <section class="container">
            <a-layout-content>
                <div class="content content-bind-mobile">
                    <div id="components-form-bind-mobile">
                        <p class="components-form-bind-mobile"> {{$t('BindMobile.item-1')}} </p>
                        <a-form
                                :form="form"
                                :layout="formLayout"
                                class=""
                                @submit="handleSubmit">
                            <a-form-item has-feedback
                                         colon>
                                <span slot="label"> {{$t('BindMobile.item-2')}} </span>
                                <a-input
                                        :placeholder="$t('BindMobile.item-3')"
                                        type="text"
                                        v-decorator="['phone', {rules: [{ validator:checkForm }],}]"
                                        style="width: 100%">
                                    <a-select
                                            slot="addonBefore"
                                            v-decorator="['prefix',{ initialValue: '86' }]"
                                            style="width: 0.86rem">
                                        <a-select-option value="86">
                                            +86
                                        </a-select-option>
                                        <a-select-option value="87">
                                            +87
                                        </a-select-option>
                                    </a-select>
                                </a-input>
                            </a-form-item>
                            <a-form-item
                                    :label="$t('BindMobile.item-4')">
                                <a-row class="bind-mobile-check">
                                    <a-input :placeholder="$t('BindMobile.item-4')"
                                             v-decorator="['captcha',{rules: [{ validator:checkForm }]}]"/>
                                    <a-button
                                            @click="checkFun"
                                            class="captcha-bt">
                                        {{ getMobileCode() }}
                                    </a-button>
                                </a-row>
                            </a-form-item>
                            <a-form-item>
                                <a-button
                                        @click="handleSubmit"
                                        type="primary"
                                        html-type="submit"
                                        class="bind-mobile-button">
                                    {{ $t('BindMobile.item-7') }}
                                </a-button>
                            </a-form-item>
                        </a-form>
                    </div>
                </div>
            </a-layout-content>
        </section>
        <fonter/>
    </a-layout>
</template>

<script>
    import FsHead from '~/components/mobile/FangseaHeader'
    import fonter from '~/components/mobile/Footer'
    import detector from '~/assets/utils/validate'
    import tool from '~/assets/utils/tool'
    import axios from '~/plugins/axios'

    export default {
        data () {
            return {
                key: '1',
                drawer: false,
                type: 2,
                formLayout: 'vertical',
                // 定时器
                wait_timer: false,
                centerKey: 0
            }
        },
        /**
        *   初始化form
        * */
        beforeCreate () {
            this.form = this.$form.createForm(this)
        },
        methods: {
            /**
             * form 表单提交
             * @param e 提交事件
             */
            handleSubmit (e) {
                e.preventDefault()
                this.form.validateFields((err, values) => {
                    if (!err) {
                        // 点击确认只需要校验验证码成功则绑定成功，失败则提示验证码校验失败
                        // 提交邮箱账号
                        // 检测验证码是否正确
                        console.log(values)
                        const params = {
                            data: {
                                account: this.form.getFieldValue('phone'),
                                verifyCode: this.form.getFieldValue('captcha')
                            }
                        }
                        axios.post('/api/v2/user/bind', {params})
                            .then(res => {
                                if (res.data.code.toString() === '0000') {
                                    this.$message.success(res.data.msg)
                                    // 跳转
                                    this.nextStep()
                                } else {
                                    this.$message.error(this.translation(res.code.toString()))
                                }
                            }).catch(err => {
                                this.$message.error(err.message)
                            })
                    }
                })
            },
            /**
             * @param rule 检测对象
             * @param value 检测的值
             * @param callback 回调
             */
            checkForm (rule, value, callback) {
                switch (rule.fullField) {
                case 'phone' :
                    if (detector.phone(value)) {
                        callback()
                    }
                    // eslint-disable-next-line
                        callback('请填写你的手机号码！')
                    break
                case 'captcha' :
                    if (value && value.replace(/(^\s*)|(\s*$)/g, '') !== '') {
                        callback()
                    }
                    // eslint-disable-next-line
                        callback('请填写你的验证码！')
                    break
                default:
                    callback()
                }
            },
            /**
             *  初始化form状态
             *  @param e 表单对象
             * */
            changeTabs (e) {
                this.form.resetFields()
            },
            /**
             *  校验邮箱账号以及发送验证码
             * **/
            checkFun () {
                if (detector.phone(this.form.getFieldValue('phone'))) {
                    axios.post('/api/v2/user/account/bind/check', {
                        traceId: tool.UUID(),
                        data: this.form.getFieldValue('phone')
                    })
                        .then(res => {
                            if (res.data.code.toString() === '0000') {
                                // 校验成功
                                // ( 需要接受账号 )邮箱账号验证成功并发送验证码
                                this.captchaGet(this.form.getFieldValue('phone'))
                            } else {
                                this.$message.error(this.translation(res.code.toString()))
                            }
                        }).catch(err => {
                            this.$message.error(err.message)
                        })
                }
            },
            /**
             * 修改按钮的文字
             * */
            getMobileCode () {
                if (this.wait_timer > 0) {
                    return this.wait_timer + ' 秒后获取'
                }
                if (this.wait_timer === 0) {
                    this.disabled = false
                    return '重新获取'
                }
                if (this.wait_timer === false) {
                    return '获取验证码'
                }
            },
            /**
             *  下一步事件
             * **/
            nextStep () {
                this.$router.push({name: 'mobile-user-UserCenter-MyMessage'})
            },
            /**
             * 发送验证码
             * */
            captchaGet (account) {
                // 防止重复点击
                if (this.wait_timer > 0) {
                    return false
                }
                // 定时器
                this.wait_timer = 60
                let that = this
                let timerInterval = setInterval(function () {
                    if (that.wait_timer > 0) {
                        that.wait_timer--
                    } else {
                        clearInterval(timerInterval)
                    }
                }, 1000)
                //  发送请求
                axios.post('/api/v2/user/verifiycode/get', {
                    traceId: tool.UUID(),
                    data: {account: account, business: 'BIND'}
                })
                    .then(res => {
                        if (res.data.code.toString() === '0000') {
                            this.$message.success(res.data.msg)
                        }
                    })
                    .catch(err => {
                        this.$message.error(err.message)
                    })
                // 修改按钮的文字
                this.getMobileCode()
                // 改变按钮状态
                this.disabled = true
            },
            // 开启右侧抽屉导航
            openDrawer () {
                this.drawer = true
            },
            // 关闭右侧抽屉导航
            closeDrawer () {
                this.drawer = false
            }
        },
        components: {
            FsHead,
            fonter
        }
    }
</script>

<style lang="less">
    .content-bind-mobile {
        padding-top: 0.65rem;
        min-height: calc(100vh - 1.26rem);
    }
    #components-form-bind-mobile {
        max-width: 5.52rem;
        margin: 0 auto;

        .ant-tabs-bar {
            border: none;
        }

        .ant-tabs-tab {
            font-size: 16px;
        }

        .login-form-forgot {
            float: right;
            margin-top: 0.14rem;
        }

        .bind-mobile-check {
            display: flex;
            justify-content: space-between;
            input {
                margin-right: 0.24rem;
            }
        }

        .bind-mobile-button {
            width: 100%;
            font-size: 16px;
            color:rgba(255,255,255,1)!important;
        }

        .components-form-bind-mobile {
            margin-bottom: 0.32rem;
            font-size: 24px;
            color: rgba(0, 0, 0, 0.85);
        }

        .ant-input-group-addon {
            font-size: 16px;
            padding: 0.08rem 0.12rem;
        }

        a, a:hover, a:focus, a:visited {
            color: #2EA9DF;
            font-size: 14px;
        }

        .ant-form-item-label {
            color: rgba(0, 0, 0, 0.85);
        }

        .ant-input{
            font-size:16px;
            padding: 0.08rem 0.12rem;
            height: 0.4rem;
            color:#000000;
        }

        .ant-btn {
            padding: 0.08rem 0.16rem;
            height: 0.4rem;
            font-size: 16px;
        }

        .ant-form-text {
            font-size: 12px;
            color: rgba(0, 0, 0, 0.45);
        }

        .captcha-bt {
            font-size: 16px;
            color: #2EA9DF;
            border: 1px solid rgba(46, 169, 223, 1);;
        }

        input::-webkit-input-placeholder {
            color:#D9D9D9;
        }
    }
</style>
