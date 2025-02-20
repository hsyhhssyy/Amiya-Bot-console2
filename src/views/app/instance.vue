<template>
    <div class="instance-manage">
        <v-table ref="table" :load="loadList">
            <template #header>
                <el-button type="primary" @click="addBot">添加实例</el-button>
            </template>
            <template #cell="{ value, field, row }">
                <template v-if="field === 'private'">
                    <el-tag v-if="row.adapter === 'tencent'" :type="value ? '' : 'success'">
                        {{ value ? '私域' : '公域' }}
                    </el-tag>
                    <el-tag v-else type="warning">群聊</el-tag>
                </template>
                <el-tag v-if="field === 'is_main'" :type="value ? 'success' : 'info'">
                    {{ value ? '是' : '否' }}
                </el-tag>
                <el-tag v-if="field === 'is_start'" :type="value ? 'success' : 'info'">
                    {{ value ? '是' : '否' }}
                </el-tag>
            </template>
            <template #operations="{ row }">
                <el-link :underline="false" type="primary" @click="editBot(row)">编辑</el-link>
                <el-link :underline="false" type="warning" v-if="row.running" @click="closeBot(row)">关闭</el-link>
                <el-link :underline="false" type="success" v-else @click="runBot(row)">运行</el-link>
                <el-link :underline="false" type="danger" @click="deleteBot(row)">删除</el-link>
            </template>
        </v-table>

        <v-form-dialog title="编辑实例" :form="form" ref="dialog">
            <el-form-item label="APP ID">
                <el-input v-model="form.appid" placeholder="实例的 app id"/>
            </el-form-item>
            <el-form-item label="TOKEN">
                <el-input v-model="form.token" placeholder="实例的 token"/>
            </el-form-item>
            <el-form-item label="适配器类型">
                <el-select v-model="form.adapter">
                    <el-option :label="name" :value="item" v-for="(name, item) in adapterType" :key="item"/>
                </el-select>
            </el-form-item>
            <el-form-item label="随启动开启">
                <el-switch v-model="form.is_start" :active-value="1" :inactive-value="0"/>
            </el-form-item>
            <el-form-item label="可控实例">
                <el-switch v-model="form.is_main" :active-value="1" :inactive-value="0"/>
            </el-form-item>
            <el-form-item label="控制台群组ID" v-if="form.is_main">
                <el-input v-model="form.console_channel" placeholder="控制实例的群组"/>
            </el-form-item>
            <template v-if="form.adapter === 'tencent'">
                <el-divider content-position="left">QQ-Bot 配置</el-divider>
                <el-form-item label="属性">
                    <el-radio-group v-model="form.private">
                        <el-radio :label="0">公域</el-radio>
                        <el-radio :label="1">私域</el-radio>
                    </el-radio-group>
                </el-form-item>
            </template>
            <template v-if="form.adapter === 'mirai_api_http'">
                <el-divider content-position="left">Mirai-api-http 配置</el-divider>
                <el-form-item label="Host地址">
                    <el-input v-model="form.mah_host" placeholder="MAH服务的地址，本地默认为 127.0.0.1"/>
                </el-form-item>
                <el-form-item label="HTTP端口">
                    <el-input v-model="form.mah_http_port" placeholder="MAH服务的HTTP端口"/>
                </el-form-item>
                <el-form-item label="WS端口">
                    <el-input v-model="form.mah_ws_port" placeholder="MAH服务的Websocket端口"/>
                </el-form-item>
            </template>
            <template v-if="form.adapter === 'cq_http'">
                <el-divider content-position="left">CQ-Http 配置</el-divider>
                <el-form-item label="Host地址">
                    <el-input v-model="form.cq_host" placeholder="CQHTTP服务的地址，本地默认为 127.0.0.1"/>
                </el-form-item>
                <el-form-item label="HTTP端口">
                    <el-input v-model="form.cq_http_port" placeholder="CQHTTP服务的HTTP端口"/>
                </el-form-item>
                <el-form-item label="WS端口">
                    <el-input v-model="form.cq_ws_port" placeholder="CQHTTP服务的Websocket端口"/>
                </el-form-item>
            </template>
            <template #footer>
                <el-button type="primary" @click="submit(0)">保存</el-button>
                <el-button type="success" @click="submit(1)" v-if="!form.running">保存并启动</el-button>
            </template>
        </v-form-dialog>
    </div>
</template>

<script lang="ts">
import { Options, Vue } from 'vue-class-component'
import { getInstanceList, submitInstance, runInstance, closeInstance, deleteInstance } from '@/request/instance'
import Common, { StringDict } from '@/lib/common'

import VTable from '@/components/table/v-table.vue'
import VFormDialog from '@/components/v-form-dialog.vue'

@Options({
    components: {
        VTable,
        VFormDialog
    },
    computed: {
        table () {
            return this.$refs.table
        },
        dialog () {
            return this.$refs.dialog
        }
    },
    mounted () {
        this.table.setColumns({
            appid: 'APP ID',
            running: {
                title: '运行状态',
                format: (row: any, value: any) => {
                    return `<span class="status status-${value.toString()}">${value ? '运行中' : '关闭'}</span>`
                }
            },
            alive: {
                title: '在线状态',
                format: (row: any, value: any) => {
                    return `<span class="status status-${value.toString()}">${value ? '在线' : '离线'}</span>`
                }
            },
            private: '属性',
            is_start: '随启动开启',
            is_main: '可控实例',
            console_channel: '控制台群组ID',
            adapter: {
                title: '适配器类型',
                format: (row: any, value: any) => {
                    return this.adapterType[value]
                }
            }
        })
    }
})
export default class Instance extends Vue {
    table!: VTable
    dialog!: VFormDialog

    public adapterType = {
        tencent: 'QQ-Bot',
        kook: 'KOOK',
        mirai_api_http: 'Mirai-api-http',
        cq_http: 'CQ-Http'
    }

    public form = {}
    public formType = 'add'

    public async loadList () {
        const res = await getInstanceList()
        if (res) {
            this.table.setData(res.data)
        }
    }

    public addBot () {
        this.form = {
            id: 0,
            appid: '',
            token: '',
            private: 0,
            adapter: 'tencent',
            is_main: 0,
            is_start: 1,
            mah_host: '',
            mah_http_port: 0,
            mah_ws_port: 0,
            cq_host: '',
            cq_http_port: 0,
            cq_ws_port: 0
        }
        this.formType = 'add'
        this.dialog.show()
    }

    public editBot (item: StringDict) {
        this.form = Common.deepCopy(item)
        this.formType = 'edit'
        this.dialog.show()
    }

    public async closeBot (item: StringDict) {
        const res = await closeInstance(item)
        if (res) {
            await this.table.executeLoad()
        }
    }

    public async runBot (item: StringDict) {
        const res = await runInstance(item)
        if (res) {
            await this.table.executeLoad()
        }
    }

    public async deleteBot (item: StringDict) {
        const res = await deleteInstance(item)
        if (res) {
            await this.table.executeLoad()
        }
    }

    public async submit (start: number) {
        const res = await submitInstance({
            ...this.form,
            start
        })
        if (res) {
            await this.table.executeLoad()
            this.dialog.hide()
        }
    }
}
</script>

<style lang="scss">
.instance-manage .status {
    &:before {
        content: '';
        display: inline-block;
        width: 10px;
        height: 10px;
        border-radius: 50%;
        background: #67c23a;
        margin-right: 5px;
    }

    &.status-0:before {
        background: #9e9e9e;
    }
}
</style>
