<template>
    <el-drawer v-model="dialogVisiable" :destroy-on-close="true" :close-on-click-modal="false" size="30%">
        <template #header>
            <DrawerHeader :header="$t('database.databaseConnInfo')" :back="handleClose" />
        </template>
        <el-form @submit.prevent v-loading="loading" ref="formRef" :model="form" label-position="top">
            <el-row type="flex" justify="center" v-if="form.from === 'local'">
                <el-col :span="22">
                    <el-form-item :label="$t('database.containerConn')">
                        <el-tag>
                            {{ form.serviceName + ':3306' }}
                        </el-tag>
                        <el-button @click="onCopy(form.serviceName + ':3306')" icon="DocumentCopy" link></el-button>
                        <span class="input-help">
                            {{ $t('database.containerConnHelper') }}
                        </span>
                    </el-form-item>
                    <el-form-item :label="$t('database.remoteConn')">
                        <el-tag>{{ form.systemIP + ':' + form.port }}</el-tag>
                        <span class="input-help">{{ $t('database.remoteConnHelper2') }}</span>
                    </el-form-item>

                    <el-divider border-style="dashed" />

                    <el-form-item :label="$t('database.remoteAccess')" prop="privilege">
                        <el-switch v-model="form.privilege" @change="onSaveAccess" />
                        <span class="input-help">{{ $t('database.remoteConnHelper') }}</span>
                    </el-form-item>
                    <el-form-item :label="$t('database.rootPassword')" :rules="Rules.paramComplexity" prop="password">
                        <el-input type="password" show-password clearable v-model="form.password">
                            <template #append>
                                <el-button @click="onCopy(form.password)">{{ $t('commons.button.copy') }}</el-button>
                                <el-divider direction="vertical" />
                                <el-button @click="random">
                                    {{ $t('commons.button.random') }}
                                </el-button>
                            </template>
                        </el-input>
                    </el-form-item>
                </el-col>
            </el-row>
            <el-row type="flex" justify="center" v-if="form.from !== 'local'">
                <el-col :span="22">
                    <el-form-item :label="$t('database.remoteConn')">
                        <el-tag>{{ form.remoteIP + ':' + form.port }}</el-tag>
                    </el-form-item>
                    <el-form-item :label="$t('commons.login.username')">
                        <el-tag>{{ form.username }}</el-tag>
                    </el-form-item>
                    <el-form-item :label="$t('commons.login.password')">
                        <el-tag>{{ form.password }}</el-tag>
                    </el-form-item>
                </el-col>
            </el-row>
        </el-form>

        <ConfirmDialog ref="confirmDialogRef" @confirm="onSubmit" @cancel="loadPassword"></ConfirmDialog>
        <ConfirmDialog ref="confirmAccessDialogRef" @confirm="onSubmitAccess" @cancel="loadAccess"></ConfirmDialog>

        <template #footer>
            <span class="dialog-footer">
                <el-button :disabled="loading" @click="dialogVisiable = false">
                    {{ $t('commons.button.cancel') }}
                </el-button>
                <el-button :disabled="loading" type="primary" @click="onSave(formRef)">
                    {{ $t('commons.button.confirm') }}
                </el-button>
            </span>
        </template>
    </el-drawer>
</template>

<script lang="ts" setup>
import { reactive, ref } from 'vue';
import { Rules } from '@/global/form-rules';
import i18n from '@/lang';
import { ElForm } from 'element-plus';
import { getRemoteDB, loadRemoteAccess, updateMysqlAccess, updateMysqlPassword } from '@/api/modules/database';
import ConfirmDialog from '@/components/confirm-dialog/index.vue';
import { GetAppConnInfo } from '@/api/modules/app';
import DrawerHeader from '@/components/drawer-header/index.vue';
import { MsgError, MsgSuccess } from '@/utils/message';
import { getRandomStr } from '@/utils/util';
import { getSettingInfo } from '@/api/modules/setting';
import useClipboard from 'vue-clipboard3';
const { toClipboard } = useClipboard();

const loading = ref(false);

const dialogVisiable = ref(false);
const form = reactive({
    systemIP: '',
    password: '',
    serviceName: '',
    privilege: false,
    port: 0,

    from: '',
    username: '',
    remoteIP: '',
});

const confirmDialogRef = ref();
const confirmAccessDialogRef = ref();

type FormInstance = InstanceType<typeof ElForm>;
const formRef = ref<FormInstance>();

interface DialogProps {
    from: string;
    remoteIP: string;
}

const acceptParams = (param: DialogProps): void => {
    form.password = '';
    form.from = param.from;
    if (form.from === 'local') {
        loadAccess();
    }
    loadPassword();
    dialogVisiable.value = true;
};

const random = async () => {
    form.password = getRandomStr(16);
};

const onCopy = async (value: string) => {
    try {
        await toClipboard(value);
        MsgSuccess(i18n.global.t('commons.msg.copySuccess'));
    } catch (e) {
        MsgError(i18n.global.t('commons.msg.copyfailed'));
    }
};

const handleClose = () => {
    dialogVisiable.value = false;
};

const loadAccess = async () => {
    const res = await loadRemoteAccess();
    form.privilege = res.data;
};

const loadSystemIP = async () => {
    const res = await getSettingInfo();
    form.systemIP = res.data.systemIP || i18n.global.t('database.localIP');
};

const loadPassword = async () => {
    if (form.from === 'local') {
        const res = await GetAppConnInfo('mysql');
        form.password = res.data.password || '';
        form.port = res.data.port || 3306;
        form.serviceName = res.data.serviceName || '';
        loadSystemIP();
        return;
    }
    const res = await getRemoteDB(form.from);
    form.password = res.data.password || '';
    form.port = res.data.port || 3306;
    form.username = res.data.username;
    form.password = res.data.password;
    form.remoteIP = res.data.address;
};

const onSubmit = async () => {
    let param = {
        id: 0,
        from: form.from,
        value: form.password,
    };
    loading.value = true;
    await updateMysqlPassword(param)
        .then(() => {
            loading.value = false;
            MsgSuccess(i18n.global.t('commons.msg.operationSuccess'));
            dialogVisiable.value = false;
        })
        .catch(() => {
            loading.value = false;
        });
};

const onSave = async (formEl: FormInstance | undefined) => {
    if (!formEl) return;
    formEl.validate(async (valid) => {
        if (!valid) return;
        let params = {
            header: i18n.global.t('database.confChange'),
            operationInfo: i18n.global.t('database.restartNowHelper'),
            submitInputInfo: i18n.global.t('database.restartNow'),
        };
        confirmDialogRef.value!.acceptParams(params);
    });
};

const onSubmitAccess = async () => {
    let param = {
        id: 0,
        from: form.from,
        value: form.privilege ? '%' : 'localhost',
    };
    loading.value = true;
    await updateMysqlAccess(param)
        .then(() => {
            loading.value = false;
            MsgSuccess(i18n.global.t('commons.msg.operationSuccess'));
            dialogVisiable.value = false;
        })
        .catch(() => {
            loading.value = false;
        });
};

const onSaveAccess = () => {
    let params = {
        header: i18n.global.t('database.confChange'),
        operationInfo: i18n.global.t('database.restartNowHelper'),
        submitInputInfo: i18n.global.t('database.restartNow'),
    };
    confirmAccessDialogRef.value!.acceptParams(params);
};

defineExpose({
    acceptParams,
});
</script>
