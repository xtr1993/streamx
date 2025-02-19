<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<template>
  <LoginFormTitle v-show="getShow" class="enter-x mb-40px text-light-50" />
  <Form
    class="p-4 enter-x signin-form"
    :model="formData"
    :rules="getFormRules"
    ref="formRef"
    v-show="getShow"
    @keypress.enter="handleLogin"
  >
    <FormItem name="account" class="enter-x">
      <Input
        v-model:value="formData.account"
        :placeholder="t('sys.login.userName')"
        class="fix-auto-fill">
        <template #prefix>
          <user-outlined type="user" />
        </template>
      </Input>
    </FormItem>
    <FormItem name="password" class="enter-x">
      <InputPassword
        visibilityToggle
        v-model:value="formData.password"
        :placeholder="t('sys.login.password')">
        <template #prefix>
          <lock-outlined type="user" />
        </template>
      </InputPassword>
    </FormItem>
    <FormItem class="enter-x signin-btn">
      <Button type="primary" block @click="handleLogin" :loading="loading">
        {{ loginText.buttonText }}
      </Button>
    </FormItem>
    <FormItem class="enter-x text-left">
      <Button type="link" @click="changeLoginType"> {{ loginText.linkText }} </Button>
    </FormItem>
  </Form>
  <TeamModal v-model:visible="modelVisible" :userId="userId" @success="handleTeamSuccess" />
</template>
<script lang="ts" setup>
  import { reactive, ref, unref, computed } from 'vue';
  import { UserOutlined, LockOutlined } from '@ant-design/icons-vue';

  import { Form, Input, Button } from 'ant-design-vue';
  import LoginFormTitle from './LoginFormTitle.vue';

  import { useI18n } from '/@/hooks/web/useI18n';
  import { useMessage } from '/@/hooks/web/useMessage';

  import { useUserStore } from '/@/store/modules/user';
  import {
    LoginStateEnum,
    useLoginState,
    useFormRules,
    useFormValid,
    LoginTypeEnum,
  } from './useLogin';
  import { useDesign } from '/@/hooks/web/useDesign';
  import { loginApi, loginLadpApi } from '/@/api/system/user';
  import { APP_TEAMID_KEY_ } from '/@/enums/cacheEnum';
  import TeamModal from './teamModal.vue';
  import { fetchUserTeam } from '/@/api/system/member';
  import { LoginResultModel } from '/@/api/system/model/userModel';
  import { Result } from '/#/axios';
  const FormItem = Form.Item;
  const InputPassword = Input.Password;

  const { t } = useI18n();
  const { createErrorModal, createMessage } = useMessage();
  const { prefixCls } = useDesign('login');
  const userStore = useUserStore();
  const { getLoginState } = useLoginState();
  const { getFormRules } = useFormRules();
  interface LoginForm {
    account: string;
    password: string;
  }
  const formRef = ref();
  const loading = ref(false);
  const userId = ref('');
  const modelVisible = ref(false);
  const loginType = ref(LoginTypeEnum.LOCAL);
  const formData = reactive<LoginForm>({
    account: '',
    password: '',
  });

  const loginText = computed(() => {
    const localText = 'Login';
    const ldapText = 'openLDAP';
    if (loginType.value === LoginTypeEnum.LOCAL) {
      return { buttonText: localText, linkText: 'Login by openLDAP' };
    }
    return { buttonText: ldapText, linkText: 'Login by Password' };
  });

  const { validForm } = useFormValid(formRef);

  const getShow = computed(() => unref(getLoginState) === LoginStateEnum.LOGIN);

  async function handleLogin() {
    try {
      const loginFormValue = await validForm();
      if (!loginFormValue) return;
      await handleLoginAction(loginFormValue);
    } catch (error) {
      console.error(error);
    }
  }
  async function handleLoginRequest(loginFormValue: LoginForm): Promise<Result<LoginResultModel>> {
    // local login
    if (loginType.value == LoginTypeEnum.LOCAL) {
      const { data } = await loginApi(
        { password: loginFormValue.password, username: loginFormValue.account },
        'none',
      );
      return data;
    }
    const { data } = await loginLadpApi(
      { password: loginFormValue.password, username: loginFormValue.account },
      'none',
    );
    return data;
  }
  async function handleLoginAction(loginFormValue: LoginForm) {
    try {
      loading.value = true;
      try {
        const { code, data } = await handleLoginRequest(loginFormValue);

        if (code != null && code != undefined) {
          if (code == 0 || code == 1) {
            const message =
              'SignIn failed,' +
              (code === 0 ? ' authentication error' : ' current User is locked.');
            createMessage.error(message);
            return;
          } else if (code == 403) {
            userId.value = data as unknown as string;
            const teamList = await fetchUserTeam({ userId: userId.value });
            userStore.setTeamList(teamList.map((i) => ({ label: i.teamName, value: i.id })));
            modelVisible.value = true;
            return;
          } else {
            console.log(data);
          }
        }
        userStore.setData(data);
        let successText = t('sys.login.loginSuccessDesc');
        if (data?.user) {
          const { teamId, nickName } = data.user;
          userStore.teamId = teamId || '';
          sessionStorage.setItem(APP_TEAMID_KEY_, teamId || '');
          localStorage.setItem(APP_TEAMID_KEY_, teamId || '');
          successText += `: ${nickName}`;
        }

        const loginSuccess = await userStore.afterLoginAction(true);
        if (loginSuccess) {
          createMessage.success(`${t('sys.login.loginSuccessTitle')} ${successText}`);
        }
      } catch (error: any) {
        createMessage.error(error.response?.data?.data?.message || 'login failed');
        return Promise.reject(error);
      }
    } catch (error) {
      createErrorModal({
        title: t('sys.api.errorTip'),
        content: (error as unknown as Error).message || t('sys.api.networkExceptionMsg'),
        getContainer: () => document.body.querySelector(`.${prefixCls}`) || document.body,
      });
    } finally {
      loading.value = false;
    }
  }

  function handleTeamSuccess() {
    modelVisible.value = false;
    handleLogin();
  }
  function changeLoginType() {
    if (loginType.value === LoginTypeEnum.LOCAL) {
      loginType.value = LoginTypeEnum.LDAP;
      return;
    }
    loginType.value = LoginTypeEnum.LOCAL;
  }
</script>
