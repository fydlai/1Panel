<template>
    <el-drawer
        :close-on-click-modal="false"
        v-model="open"
        :title="$t('app.install')"
        size="50%"
        :before-close="handleClose"
    >
        <template #header>
            <Header :header="$t('app.install')" :back="handleClose"></Header>
        </template>
        <el-row v-loading="loading">
            <el-col :span="22" :offset="1">
                <el-alert :title="$t('app.appInstallWarn')" class="common-prompt" :closable="false" type="error" />
                <el-form
                    @submit.prevent
                    ref="paramForm"
                    label-position="top"
                    :model="req"
                    label-width="150px"
                    :rules="rules"
                    :validate-on-rule-change="false"
                >
                    <el-form-item :label="$t('commons.table.name')" prop="name">
                        <el-input v-model.trim="req.name"></el-input>
                    </el-form-item>
                    <Params
                        v-if="open"
                        v-model:form="req.params"
                        v-model:params="installData.params"
                        v-model:rules="rules.params"
                        :propStart="'params.'"
                    ></Params>
                    <el-form-item prop="advanced">
                        <el-checkbox v-model="req.advanced" :label="$t('app.advanced')" size="large" />
                    </el-form-item>
                    <div v-if="req.advanced">
                        <el-form-item :label="$t('app.containerName')" prop="containerName">
                            <el-input
                                v-model.trim="req.containerName"
                                :placeholder="$t('app.containerNameHelper')"
                            ></el-input>
                        </el-form-item>
                        <el-form-item
                            :label="$t('container.cpuQuota')"
                            prop="cpuQuota"
                            :rules="checkNumberRange(0, limits.cpu)"
                        >
                            <el-input type="number" style="width: 40%" v-model.number="req.cpuQuota" maxlength="5">
                                <template #append>{{ $t('app.cpuCore') }}</template>
                            </el-input>
                            <span class="input-help">
                                {{ $t('container.limitHelper', [limits.cpu]) }}{{ $t('commons.units.core') }}
                            </span>
                        </el-form-item>
                        <el-form-item
                            :label="$t('container.memoryLimit')"
                            prop="memoryLimit"
                            :rules="checkNumberRange(0, limits.memory)"
                        >
                            <el-input style="width: 40%" v-model.number="req.memoryLimit" maxlength="10">
                                <template #append>
                                    <el-select
                                        v-model="req.memoryUnit"
                                        placeholder="Select"
                                        style="width: 85px"
                                        @change="changeUnit"
                                    >
                                        <el-option label="MB" value="M" />
                                        <el-option label="GB" value="G" />
                                    </el-select>
                                </template>
                            </el-input>
                            <span class="input-help">
                                {{ $t('container.limitHelper', [limits.memory]) }}{{ req.memoryUnit }}B
                            </span>
                        </el-form-item>
                        <el-form-item prop="allowPort" v-if="canEditPort(installData.app.key)">
                            <el-checkbox v-model="req.allowPort" :label="$t('app.allowPort')" size="large" />
                            <span class="input-help">{{ $t('app.allowPortHelper') }}</span>
                        </el-form-item>
                        <el-form-item prop="editCompose">
                            <el-checkbox v-model="req.editCompose" :label="$t('app.editCompose')" size="large" />
                            <span class="input-help">{{ $t('app.editComposeHelper') }}</span>
                        </el-form-item>
                        <div v-if="req.editCompose">
                            <codemirror
                                :autofocus="true"
                                placeholder=""
                                :indent-with-tab="true"
                                :tabSize="4"
                                style="height: 400px"
                                :lineWrapping="true"
                                :matchBrackets="true"
                                theme="cobalt"
                                :styleActiveLine="true"
                                :extensions="extensions"
                                v-model="req.dockerCompose"
                            />
                        </div>
                    </div>
                </el-form>
            </el-col>
        </el-row>

        <template #footer>
            <span class="dialog-footer">
                <el-button @click="handleClose" :disabled="loading">{{ $t('commons.button.cancel') }}</el-button>
                <el-button type="primary" @click="submit(paramForm)" :disabled="loading">
                    {{ $t('commons.button.confirm') }}
                </el-button>
            </span>
        </template>
    </el-drawer>
</template>

<script lang="ts" setup name="appInstall">
import { App } from '@/api/interface/app';
import { InstallApp } from '@/api/modules/app';
import { Rules, checkNumberRange } from '@/global/form-rules';
import { canEditPort } from '@/global/business';
import { FormInstance, FormRules } from 'element-plus';
import { onMounted, reactive, ref } from 'vue';
import { useRouter } from 'vue-router';
import Params from '../params/index.vue';
import Header from '@/components/drawer-header/index.vue';
import { Codemirror } from 'vue-codemirror';
import { javascript } from '@codemirror/lang-javascript';
import { oneDark } from '@codemirror/theme-one-dark';
import i18n from '@/lang';
import { MsgError } from '@/utils/message';
import { Container } from '@/api/interface/container';
import { loadResourceLimit } from '@/api/modules/container';

const extensions = [javascript(), oneDark];
const router = useRouter();

interface InstallRrops {
    appDetailId: number;
    params?: App.AppParams;
    app: any;
    compose: string;
}

const installData = ref<InstallRrops>({
    appDetailId: 0,
    app: {},
    compose: '',
});
const open = ref(false);
const rules = ref<FormRules>({
    name: [Rules.appName],
    params: [],
    containerName: [Rules.containerName],
    cpuQuota: [Rules.requiredInput, checkNumberRange(0, 99999)],
    memoryLimit: [Rules.requiredInput, checkNumberRange(0, 9999999999)],
});
const loading = ref(false);
const paramForm = ref<FormInstance>();
const form = ref<{ [key: string]: any }>({});
const initData = () => ({
    appDetailId: 0,
    params: form.value,
    name: '',
    advanced: true,
    cpuQuota: 0,
    memoryLimit: 0,
    memoryUnit: 'M',
    containerName: '',
    allowPort: false,
    editCompose: false,
    dockerCompose: '',
});
const req = reactive(initData());
const limits = ref<Container.ResourceLimit>({
    cpu: null as number,
    memory: null as number,
});
const oldMemory = ref<number>(0);

const handleClose = () => {
    open.value = false;
    resetForm();
};

const changeUnit = () => {
    if (req.memoryUnit == 'M') {
        limits.value.memory = oldMemory.value;
    } else {
        limits.value.memory = Number((oldMemory.value / 1024).toFixed(2));
    }
};

const resetForm = () => {
    if (paramForm.value) {
        paramForm.value.clearValidate();
        paramForm.value.resetFields();
    }
    Object.assign(req, initData());
    req.dockerCompose = installData.value.compose;
};

const acceptParams = (props: InstallRrops): void => {
    installData.value = props;
    resetForm();
    req.name = props.app.key;
    open.value = true;
};

const submit = async (formEl: FormInstance | undefined) => {
    if (!formEl) return;
    await formEl.validate((valid) => {
        if (!valid) {
            return;
        }
        if (req.editCompose && req.dockerCompose == '') {
            MsgError(i18n.global.t('app.composeNullErr'));
            return;
        }
        req.appDetailId = installData.value.appDetailId;
        if (req.cpuQuota < 0) {
            req.cpuQuota = 0;
        }
        if (req.memoryLimit < 0) {
            req.memoryLimit = 0;
        }
        if (canEditPort(installData.value.app.key) && !req.allowPort) {
            ElMessageBox.confirm(i18n.global.t('app.installWarn'), i18n.global.t('app.checkTitle'), {
                confirmButtonText: i18n.global.t('commons.button.confirm'),
                cancelButtonText: i18n.global.t('commons.button.cancel'),
            }).then(async () => {
                install();
            });
        } else {
            install();
        }
    });
};

const install = () => {
    loading.value = true;
    InstallApp(req)
        .then(() => {
            handleClose();
            router.push({ path: '/apps/installed' });
        })
        .finally(() => {
            loading.value = false;
        });
};

const loadLimit = async () => {
    const res = await loadResourceLimit();
    limits.value = res.data;
    limits.value.memory = Number((limits.value.memory / 1024 / 1024).toFixed(2));
    oldMemory.value = limits.value.memory;
};

defineExpose({
    acceptParams,
});

onMounted(() => {
    loadLimit();
});
</script>
