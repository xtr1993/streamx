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
  export default defineComponent({
    name: 'SystemSetting',
  });
</script>
<script setup lang="ts" name="SystemSetting">
  import { Collapse } from 'ant-design-vue';
  import { ref, onMounted, computed } from 'vue';
  import { fetchSystemSetting } from '/@/api/flink/setting';
  import { SystemSetting } from '/@/api/flink/setting/types/setting.type';
  import SettingList from './SettingList.vue';
  import { fetchSystemSettingUpdate } from '/@/api/flink/setting/alert';
  import { useMessage } from '/@/hooks/web/useMessage';

  const CollapsePane = Collapse.Panel;
  const { createMessage } = useMessage();
  const settings = ref<SystemSetting[]>([]);

  const settingsList = computed(() => {
    return [
      {
        key: 1,
        title: 'Maven Setting',
        isPassword: (item) => item.settingKey === 'streampark.maven.auth.password',
        data: settings.value.filter((i) => i.settingKey.indexOf('streampark.maven') > -1),
      },
      {
        key: 2,
        title: 'Docker Setting',
        isPassword: (item) => item.settingKey === 'docker.register.password',
        data: settings.value.filter((i) => i.settingKey.indexOf('docker.register') > -1),
      },
      {
        key: 3,
        title: 'Sender Email Setting',
        isPassword: (item) => item.settingKey === 'alert.email.password',
        data: settings.value.filter((i) => i.settingKey.indexOf('alert.email') > -1),
      },
      {
        key: 4,
        title: 'Console Setting',
        isPassword: () => false,
        data: settings.value.filter((i) => i.settingKey.indexOf('streampark.console') > -1),
      },
      {
        key: 5,
        title: 'Ingrsss Setting',
        isPassword: () => false,
        data: settings.value.filter((i) => i.settingKey.indexOf('ingrsss') > -1),
      },
    ];
  });
  const collapseActive = ref(['1', '2', '3', '4', '5']);
  /* Get all system settings */
  async function getSettingAll() {
    const res = await fetchSystemSetting();
    settings.value = res;
  }

  /* Update setting value */
  async function handleSettingUpdate(record: SystemSetting) {
    await fetchSystemSettingUpdate({
      settingKey: record.settingKey,
      settingValue: record.settingValue !== 'true',
    });
    createMessage.success('setting updated successfully');
    getSettingAll();
  }

  onMounted(() => {
    getSettingAll();
  });
</script>

<template>
  <Collapse class="collapse" v-model:activeKey="collapseActive">
    <CollapsePane v-for="item in settingsList" :key="item.key" :header="item.title">
      <div class="bg-white">
        <SettingList
          :data="item.data"
          :isPassword="item.isPassword"
          @update-value="handleSettingUpdate"
          @reload="getSettingAll"
        />
      </div>
    </CollapsePane>
  </Collapse>
</template>
