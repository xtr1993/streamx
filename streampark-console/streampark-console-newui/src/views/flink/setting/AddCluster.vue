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
<script lang="ts">
  import { defineComponent } from 'vue';
  import { useI18n } from '/@/hooks/web/useI18n';
  import { useGo } from '/@/hooks/web/usePage';
  export default defineComponent({
    name: 'AddCluster',
  });
</script>
<script setup lang="ts" name="AddCluster">
  import { PageWrapper } from '/@/components/Page';
  import { BasicForm, useForm } from '/@/components/Form';
  import { useMessage } from '/@/hooks/web/useMessage';
  import { fetchCheckCluster, fetchCreateCluster } from '/@/api/flink/setting/flinkCluster';

  import { useClusterSetting } from './hooks/useClusterSetting';

  const go = useGo();
  const { t } = useI18n();
  const { Swal } = useMessage();

  const { getLoading, changeLoading, getClusterSchema, handleSubmitParams } = useClusterSetting();

  const [registerForm, { submit }] = useForm({
    labelWidth: 120,
    colon: true,
    labelCol: { lg: { span: 5, offset: 0 }, sm: { span: 7, offset: 0 } },
    wrapperCol: { lg: { span: 16, offset: 0 }, sm: { span: 17, offset: 0 } },
    baseColProps: { span: 24 },
    showActionButtonGroup: false,
  });
  async function handleSubmitCluster(values: Recordable) {
    try {
      changeLoading(true);
      const params = handleSubmitParams(values);
      if (Object.keys(params).length > 0) {
        const res = await fetchCheckCluster(params);
        if (res === 'success') {
          const resp = await fetchCreateCluster(params);
          if (resp.status) {
            Swal.fire({
              icon: 'success',
              title: values.clusterName.concat(' create successful!'),
              showConfirmButton: false,
              timer: 2000,
            });
            go('/flink/setting?activeKey=cluster');
          } else {
            Swal.fire(resp.msg);
          }
        } else if (res === 'exists') {
          Swal.fire(
            'Failed',
            'the cluster name: ' + values.clusterName + ' is already exists,please check',
            'error',
          );
        } else {
          Swal.fire(
            'Failed',
            'the address is invalid or connection failure, please check',
            'error',
          );
        }
      }
    } catch (error) {
      console.error(error);
    } finally {
      changeLoading(false);
    }
  }
</script>
<template>
  <PageWrapper content-background content-class="py-30px">
    <BasicForm @register="registerForm" @submit="handleSubmitCluster" :schemas="getClusterSchema">
      <template #formFooter>
        <div class="flex items-center w-full justify-center">
          <a-button @click="go('/flink/setting?activeKey=cluster')">
            {{ t('common.cancelText') }}
          </a-button>
          <a-button class="ml-4" :loading="getLoading" type="primary" @click="submit()">
            {{ t('common.submitText') }}
          </a-button>
        </div>
      </template></BasicForm
    >
  </PageWrapper>
</template>
