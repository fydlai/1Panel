<template>
    <el-drawer :close-on-click-modal="false" v-model="open" size="50%">
        <template #header>
            <DrawerHeader :header="$t('website.create')" :back="handleClose">
                <template #buttons>
                    <span class="drawer-header-button">
                        <template
                            v-for="item in [
                                {
                                    label: i18n.global.t('website.deployment'),
                                    value: 'deployment',
                                },
                                {
                                    label: i18n.global.t('runtime.runtime'),
                                    value: 'runtime',
                                },

                                {
                                    label: i18n.global.t('website.proxy'),
                                    value: 'proxy',
                                },
                                {
                                    label: i18n.global.t('website.static'),
                                    value: 'static',
                                },
                            ]"
                            :key="item.value"
                        >
                            <el-button
                                :class="website.type === item.value ? 'active-button' : ''"
                                @click="changeType(item.value)"
                                :type="website.type === item.value ? '' : 'info'"
                                :plain="website.type === item.value"
                                :text="website.type !== item.value"
                                :bg="website.type !== item.value"
                            >
                                {{ item.label }}
                            </el-button>
                        </template>
                    </span>
                </template>
            </DrawerHeader>
        </template>
        <el-row v-loading="loading" :class="{ mask: !versionExist }">
            <el-col :span="22" :offset="1">
                <el-alert
                    v-if="website.type == 'deployment'"
                    :title="$t('website.websiteDeploymentHelper')"
                    type="info"
                    :closable="false"
                />
                <el-alert
                    v-if="website.type == 'static'"
                    :title="$t('website.websiteStatictHelper')"
                    type="info"
                    :closable="false"
                />
                <el-alert
                    v-if="website.type == 'proxy'"
                    :title="$t('website.websiteProxyHelper')"
                    type="info"
                    :closable="false"
                />
                <el-alert
                    v-if="website.type == 'runtime'"
                    :title="$t('website.runtimeProxyHelper')"
                    type="info"
                    :closable="false"
                />
                <br />
                <el-form
                    ref="websiteForm"
                    label-position="top"
                    :model="website"
                    label-width="125px"
                    :rules="rules"
                    :validate-on-rule-change="false"
                >
                    <el-form-item :label="$t('website.group')" prop="webSiteGroupId">
                        <el-select v-model="website.webSiteGroupId">
                            <el-option
                                v-for="(group, index) in groups"
                                :key="index"
                                :label="group.name"
                                :value="group.id"
                            ></el-option>
                        </el-select>
                    </el-form-item>
                    <div v-if="website.type === 'deployment'">
                        <el-form-item prop="appType">
                            <el-radio-group v-model="website.appType" @change="changeAppType(website.appType)">
                                <el-radio :label="'installed'" :value="'installed'">
                                    {{ $t('website.appInstalled') }}
                                </el-radio>
                                <el-radio :label="'new'">
                                    {{ $t('website.appNew') }}
                                </el-radio>
                            </el-radio-group>
                        </el-form-item>
                        <el-form-item
                            v-if="website.appType == 'installed'"
                            :label="$t('website.appInstalled')"
                            prop="appInstallId"
                        >
                            <el-select v-model="website.appInstallId">
                                <el-option
                                    v-for="(appInstall, index) in appInstalles"
                                    :key="index"
                                    :label="appInstall.name"
                                    :value="appInstall.id"
                                ></el-option>
                            </el-select>
                        </el-form-item>
                        <div v-if="website.appType == 'new'">
                            <el-form-item :label="$t('app.app')" prop="appinstall.appId">
                                <el-row :gutter="20">
                                    <el-col :span="12">
                                        <el-select v-model="website.appinstall.appId" @change="changeApp()">
                                            <el-option
                                                v-for="(app, index) in apps"
                                                :key="index"
                                                :label="app.name"
                                                :value="app.id"
                                            ></el-option>
                                        </el-select>
                                    </el-col>
                                    <el-col :span="12">
                                        <el-select
                                            v-model="website.appinstall.version"
                                            @change="getAppDetail(website.appinstall.version)"
                                        >
                                            <el-option
                                                v-for="(version, index) in appVersions"
                                                :key="index"
                                                :label="version"
                                                :value="version"
                                            ></el-option>
                                        </el-select>
                                    </el-col>
                                </el-row>
                            </el-form-item>
                            <el-form-item :label="$t('commons.table.name')" prop="appinstall.name">
                                <el-input v-model.trim="website.appinstall.name"></el-input>
                            </el-form-item>
                            <Params
                                :key="paramKey"
                                v-model:form="website.appinstall.params"
                                v-model:rules="rules.appinstall.params"
                                :params="appParams"
                                :propStart="'appinstall.params.'"
                            ></Params>
                        </div>
                    </div>
                    <div v-if="website.type === 'runtime'">
                        <el-form-item :label="$t('runtime.runtime')" prop="runtimeID">
                            <el-select
                                v-model="website.runtimeID"
                                @change="changeRuntime(website.runtimeID)"
                                filterable
                            >
                                <el-option
                                    v-for="run in runtimes"
                                    :key="run.name"
                                    :label="run.name + '(' + $t('runtime.' + run.resource) + ')'"
                                    :value="run.id"
                                ></el-option>
                            </el-select>
                        </el-form-item>
                        <Params
                            v-if="runtimeResource === 'appstore'"
                            :key="paramKey"
                            v-model:form="website.appinstall.params"
                            v-model:rules="rules.appinstall.params"
                            :params="appParams"
                            :propStart="'appinstall.params.'"
                        ></Params>
                        <div v-else>
                            <el-form-item :label="$t('website.proxyType')" prop="proxyType">
                                <el-select v-model="website.proxyType">
                                    <el-option :label="$t('website.tcp')" :value="'tcp'"></el-option>
                                    <el-option :label="$t('website.unix')" :value="'unix'"></el-option>
                                </el-select>
                            </el-form-item>
                            <el-form-item
                                v-if="website.proxyType === 'tcp'"
                                :label="$t('commons.table.port')"
                                prop="port"
                            >
                                <el-input v-model.number="website.port"></el-input>
                            </el-form-item>
                        </div>
                    </div>
                    <el-form-item
                        prop="advanced"
                        v-if="
                            website.type === 'runtime' || (website.type === 'deployment' && website.appType === 'new')
                        "
                    >
                        <el-checkbox v-model="website.appinstall.advanced" :label="$t('app.advanced')" size="large" />
                    </el-form-item>

                    <div v-if="website.appinstall.advanced">
                        <el-form-item :label="$t('app.containerName')" prop="containerName">
                            <el-input
                                v-model.trim="website.appinstall.containerName"
                                :placeholder="$t('app.containerNameHelper')"
                            ></el-input>
                        </el-form-item>
                        <el-form-item :label="$t('container.cpuQuota')" prop="appinstall.cpuQuota">
                            <el-input
                                type="number"
                                style="width: 40%"
                                v-model.number="website.appinstall.cpuQuota"
                                maxlength="5"
                            >
                                <template #append>{{ $t('app.cpuCore') }}</template>
                            </el-input>
                            <span class="input-help">{{ $t('container.limitHelper') }}</span>
                        </el-form-item>
                        <el-form-item :label="$t('container.memoryLimit')" prop="appinstall.memoryLimit">
                            <el-input style="width: 40%" v-model.number="website.appinstall.memoryLimit" maxlength="10">
                                <template #append>
                                    <el-select
                                        v-model="website.appinstall.memoryUnit"
                                        placeholder="Select"
                                        class="pre-select"
                                    >
                                        <el-option label="KB" value="K" />
                                        <el-option label="MB" value="M" />
                                        <el-option label="GB" value="G" />
                                    </el-select>
                                </template>
                            </el-input>
                            <span class="input-help">{{ $t('container.limitHelper') }}</span>
                        </el-form-item>
                        <el-form-item prop="allowPort">
                            <el-checkbox
                                v-model="website.appinstall.allowPort"
                                :label="$t('app.allowPort')"
                                size="large"
                            />
                            <span class="input-help">{{ $t('app.allowPortHelper') }}</span>
                        </el-form-item>
                    </div>
                    <el-form-item :label="$t('website.primaryDomain')" prop="primaryDomain">
                        <el-input
                            v-model.trim="website.primaryDomain"
                            @input="changeAlias(website.primaryDomain)"
                        ></el-input>
                    </el-form-item>
                    <el-form-item :label="$t('website.otherDomains')" prop="otherDomains">
                        <el-input
                            type="textarea"
                            :autosize="{ minRows: 2, maxRows: 6 }"
                            v-model="website.otherDomains"
                            :placeholder="$t('website.domainHelper')"
                        ></el-input>
                    </el-form-item>
                    <el-form-item prop="IPV6">
                        <el-checkbox v-model="website.IPV6" :label="$t('website.ipv6')" size="large" />
                    </el-form-item>
                    <el-form-item :label="$t('website.alias')" prop="alias">
                        <el-input v-model.trim="website.alias" :placeholder="$t('website.aliasHelper')"></el-input>
                        <div>
                            <span class="input-help">
                                <span>{{ $t('website.staticPath') + staticPath + website.alias }}</span>
                                <span v-if="website.type === 'static' && website.alias != ''">{{ '/index' }}</span>
                            </span>
                        </div>
                    </el-form-item>
                    <el-form-item
                        v-if="website.type === 'proxy'"
                        :label="$t('website.proxyAddress')"
                        prop="proxyAddress"
                    >
                        <el-input v-model="website.proxyAddress" :placeholder="$t('website.proxyHelper')">
                            <template #prepend>
                                <el-select v-model="website.proxyProtocol" class="pre-select">
                                    <el-option label="http" value="http://" />
                                    <el-option label="https" value="https://" />
                                    <el-option :label="$t('website.other')" value="" />
                                </el-select>
                            </template>
                        </el-input>
                    </el-form-item>
                    <el-form-item :label="$t('website.remark')" prop="remark">
                        <el-input v-model="website.remark"></el-input>
                    </el-form-item>
                </el-form>
            </el-col>
        </el-row>
        <template #footer>
            <span>
                <el-button @click="handleClose" :disabled="loading">{{ $t('commons.button.cancel') }}</el-button>
                <el-button type="primary" @click="submit(websiteForm)" :disabled="loading">
                    {{ $t('commons.button.confirm') }}
                </el-button>
            </span>
        </template>
        <Check ref="preCheckRef"></Check>
        <el-card width="30%" v-if="!versionExist" class="mask-prompt">
            <span>
                {{ $t('runtime.openrestryWarn') }}
            </span>
        </el-card>
    </el-drawer>
</template>

<script lang="ts" setup name="CreateWebSite">
import DrawerHeader from '@/components/drawer-header/index.vue';
import { App } from '@/api/interface/app';
import {
    GetApp,
    GetAppDetail,
    SearchApp,
    GetAppInstalled,
    GetAppDetailByID,
    CheckAppInstalled,
} from '@/api/modules/app';
import { CreateWebsite, PreCheck } from '@/api/modules/website';
import { Rules, checkNumberRange } from '@/global/form-rules';
import i18n from '@/lang';
import { ElForm, FormInstance } from 'element-plus';
import { reactive, ref } from 'vue';
import Params from '@/views/app-store/detail/params/index.vue';
import Check from '../check/index.vue';
import { MsgSuccess } from '@/utils/message';
import { GetGroupList } from '@/api/modules/group';
import { Group } from '@/api/interface/group';
import { SearchRuntimes } from '@/api/modules/runtime';
import { Runtime } from '@/api/interface/runtime';

const websiteForm = ref<FormInstance>();
const website = ref({
    primaryDomain: '',
    type: 'deployment',
    alias: '',
    remark: '',
    appType: 'installed',
    appInstallId: undefined,
    webSiteGroupId: 1,
    otherDomains: '',
    proxy: '',
    runtimeID: undefined,
    appinstall: {
        appId: 0,
        name: '',
        appDetailId: 0,
        params: {},
        version: '',
        appkey: '',

        advanced: false,
        cpuQuota: 0,
        memoryLimit: 0,
        memoryUnit: 'MB',
        containerName: '',
        allowPort: false,
    },
    IPV6: false,
    proxyType: 'tcp',
    port: 9000,
    proxyProtocol: 'http://',
    proxyAddress: '',
});
const rules = ref<any>({
    primaryDomain: [Rules.domain],
    alias: [Rules.linuxName],
    type: [Rules.requiredInput],
    webSiteGroupId: [Rules.requiredSelectBusiness],
    appInstallId: [Rules.requiredSelectBusiness],
    appType: [Rules.requiredInput],
    proxyAddress: [Rules.requiredInput],
    runtimeID: [Rules.requiredSelectBusiness],
    appinstall: {
        name: [Rules.appName],
        appId: [Rules.requiredSelectBusiness],
        params: {},
        cpuQuota: [Rules.requiredInput, checkNumberRange(0, 99999)],
        memoryLimit: [Rules.requiredInput, checkNumberRange(0, 9999999999)],
        containerName: [Rules.containerName],
    },
    proxyType: [Rules.requiredSelect],
    port: [Rules.port],
});

const open = ref(false);
const loading = ref(false);
const groups = ref<Group.GroupInfo[]>([]);

const appInstalles = ref<App.AppInstalled[]>([]);
const appReq = reactive({
    type: 'website',
    page: 1,
    pageSize: 20,
});
const apps = ref<App.App[]>([]);
const appVersions = ref<string[]>([]);
const appDetail = ref<App.AppDetail>();
const appParams = ref<App.AppParams>();
const paramKey = ref(1);
const preCheckRef = ref();
const staticPath = ref('');
const runtimeResource = ref('appstore');
const runtimeReq = ref<Runtime.RuntimeReq>({
    page: 1,
    pageSize: 100,
    status: 'normal',
});
const runtimes = ref<Runtime.RuntimeDTO[]>([]);
const versionExist = ref(true);
const em = defineEmits(['close']);

const handleClose = () => {
    open.value = false;
    em('close', false);
};

const changeType = (type: string) => {
    if (type == 'deployemnt') {
        if (appInstalles.value && appInstalles.value.length > 0) {
            website.value.appInstallId = appInstalles.value[0].id;
        }
    } else if (type == 'runtime') {
        checkNginxVersion();
        getRuntimes();
    } else {
        website.value.appInstallId = undefined;
    }
    website.value.type = type;
    versionExist.value = true;
};

const checkNginxVersion = async () => {
    try {
        const res = await CheckAppInstalled('openresty');
        if (res.data && res.data.version) {
            if (!compareVersions(res.data.version, '1.21.4')) {
                versionExist.value = false;
            }
        }
    } catch (error) {}
};

const searchAppInstalled = () => {
    GetAppInstalled({ type: 'website', unused: true }).then((res) => {
        appInstalles.value = res.data;
        if (res.data && res.data.length > 0) {
            website.value.appInstallId = res.data[0].id;
        }
    });
};

const searchApp = () => {
    SearchApp(appReq).then((res) => {
        apps.value = res.data.items;
        if (res.data.items.length > 0) {
            website.value.appinstall.appId = res.data.items[0].id;
            website.value.appinstall.appkey = res.data.items[0].key;
            getApp();
        }
    });
};

const changeApp = () => {
    apps.value.forEach((app) => {
        if (app.id === website.value.appinstall.appId) {
            website.value.appinstall.appkey = app.key;
            getApp();
        }
    });
};

const getApp = () => {
    GetApp(website.value.appinstall.appkey).then((res) => {
        appVersions.value = res.data.versions;
        if (res.data.versions.length > 0) {
            website.value.appinstall.version = res.data.versions[0];
            getAppDetail(res.data.versions[0]);
        }
    });
};

const getAppDetail = (version: string) => {
    GetAppDetail(website.value.appinstall.appId, version, 'app').then((res) => {
        website.value.appinstall.appDetailId = res.data.id;
        appDetail.value = res.data;
        appParams.value = res.data.params;
        paramKey.value++;
    });
};

const getAppDetailByID = (id: number) => {
    GetAppDetailByID(id).then((res) => {
        website.value.appinstall.appDetailId = res.data.id;
        appDetail.value = res.data;
        appParams.value = res.data.params;
        paramKey.value++;
    });
};

const changeRuntime = (runID: number) => {
    runtimes.value.forEach((item) => {
        if (item.id === runID) {
            runtimeResource.value = item.resource;
            if (item.resource === 'appstore') {
                getAppDetailByID(item.appDetailId);
            }
        }
    });
};

const getRuntimes = async () => {
    try {
        const res = await SearchRuntimes(runtimeReq.value);
        runtimes.value = res.data.items || [];
        if (runtimes.value.length > 0) {
            const first = runtimes.value[0];
            website.value.runtimeID = first.id;
            runtimeResource.value = first.resource;
            if (first.resource === 'appstore') {
                getAppDetailByID(first.appDetailId);
            }
        }
    } catch (error) {}
};

const acceptParams = async (installPath: string) => {
    if (websiteForm.value) {
        websiteForm.value.resetFields();
    }
    staticPath.value = installPath + '/www/sites/';

    const res = await GetGroupList({ type: 'website' });
    groups.value = res.data;
    website.value.webSiteGroupId = res.data[0].id;
    website.value.type = 'deployment';
    runtimeResource.value = 'appstore';

    searchAppInstalled();

    open.value = true;
};

const changeAppType = (type: string) => {
    if (type === 'installed') {
        searchAppInstalled();
    } else {
        searchApp();
    }
};

const submit = async (formEl: FormInstance | undefined) => {
    if (!formEl) return;
    await formEl.validate((valid) => {
        if (!valid) {
            return;
        }
        loading.value = true;
        PreCheck({})
            .then((res) => {
                if (res.data) {
                    loading.value = false;
                    preCheckRef.value.acceptParams({ items: res.data });
                } else {
                    if (website.value.type === 'proxy') {
                        website.value.proxy = website.value.proxyProtocol + website.value.proxyAddress;
                    }
                    CreateWebsite(website.value)
                        .then(() => {
                            MsgSuccess(i18n.global.t('commons.msg.createSuccess'));
                            handleClose();
                        })
                        .finally(() => {
                            loading.value = false;
                        });
                }
            })
            .catch(() => {
                loading.value = false;
            });
    });
};

const changeAlias = (value: string) => {
    website.value.alias = value;
};

function compareVersions(version1: string, version2: string): boolean {
    const v1 = version1.split('.');
    const v2 = version2.split('.');
    const len = Math.max(v1.length, v2.length);

    for (let i = 0; i < len; i++) {
        const num1 = parseInt(v1[i] || '0');
        const num2 = parseInt(v2[i] || '0');

        if (num1 !== num2) {
            return num1 > num2 ? true : false;
        }
    }

    return false;
}

defineExpose({
    acceptParams,
});
</script>
