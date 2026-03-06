<template lang="pug">
div
  h4.mb-3 推送与同步（Client & 远程宿主机）
  p.text-muted 本机与远程宿主机地址供 aw-client 双写：事件同时写入本机和宿主机；宿主机不可达时仅写本机，由 aw-sync 后续合并。Watcher 重启后生效。

  b-form-group(label="本机 server 地址" label-cols-md=3 description="Watcher 推送的主目标（通常为本机 aw-server）")
    b-input-group
      b-form-input(v-model="serverHostname" placeholder="127.0.0.1")
      b-form-input(v-model.number="serverPort" type="number" placeholder="5600" style="max-width: 6em")

  b-form-group(label="启用远程双写" label-cols-md=3 description="同时将事件推送到下方远程宿主机（best-effort，失败不影响本机）")
    b-form-checkbox(v-model="remoteEnabled" switch)

  b-form-group(v-if="remoteEnabled" label="远程宿主机地址" label-cols-md=3 description="宿主机 aw-server 的 host:port")
    b-input-group
      b-form-input(v-model="remoteHostname" placeholder="例如 192.168.1.100 或 nas.local")
      b-form-input(v-model.number="remotePort" type="number" placeholder="5600" style="max-width: 6em")
      b-input-group-append
        b-button(variant="outline-primary" :disabled="testLoading" @click="testRemote")
          | {{ testLoading ? '检测中…' : '测试连接' }}
    small.text-success(v-if="testResult === 'ok'") 连接成功
    small.text-danger(v-else-if="testResult === 'fail'") 连接失败：{{ testError }}

  hr
</template>

<script lang="ts">
import { useSettingsStore } from '~/stores/settings';
import { getClient } from '~/util/awclient';

const SERVER_HOST = 'aw_client.server_hostname' as const;
const SERVER_PORT = 'aw_client.server_port' as const;
const REMOTE_ENABLED = 'aw_client.remote_enabled' as const;
const REMOTE_HOST = 'aw_client.remote_hostname' as const;
const REMOTE_PORT = 'aw_client.remote_port' as const;

export default {
  data() {
    return {
      testLoading: false,
      testResult: '' as '' | 'ok' | 'fail',
      testError: '',
    };
  },
  computed: {
    serverHostname: {
      get(): string {
        return useSettingsStore()[SERVER_HOST] ?? '127.0.0.1';
      },
      set(v: string) {
        useSettingsStore().update({ [SERVER_HOST]: v });
      },
    },
    serverPort: {
      get(): number {
        return useSettingsStore()[SERVER_PORT] ?? 5600;
      },
      set(v: number) {
        useSettingsStore().update({ [SERVER_PORT]: v });
      },
    },
    remoteEnabled: {
      get(): boolean {
        return useSettingsStore()[REMOTE_ENABLED] ?? false;
      },
      set(v: boolean) {
        useSettingsStore().update({ [REMOTE_ENABLED]: v });
      },
    },
    remoteHostname: {
      get(): string {
        return useSettingsStore()[REMOTE_HOST] ?? '';
      },
      set(v: string) {
        useSettingsStore().update({ [REMOTE_HOST]: v });
      },
    },
    remotePort: {
      get(): number {
        return useSettingsStore()[REMOTE_PORT] ?? 5600;
      },
      set(v: number) {
        useSettingsStore().update({ [REMOTE_PORT]: v });
      },
    },
  },
  methods: {
    async testRemote() {
      const hostname = this.remoteHostname?.trim();
      const port = this.remotePort;
      if (!hostname) {
        this.testResult = 'fail';
        this.testError = '请先填写远程 hostname';
        return;
      }
      this.testLoading = true;
      this.testResult = '';
      this.testError = '';
      try {
        const client = getClient();
        const url = `/0/test-remote-server?hostname=${encodeURIComponent(hostname)}&port=${encodeURIComponent(String(port))}`;
        const res = await client.req.get(url);
        const data = res.data as { ok?: boolean; error?: string };
        if (data?.ok) {
          this.testResult = 'ok';
        } else {
          this.testResult = 'fail';
          this.testError = data?.error ?? '未知错误';
        }
      } catch (e: unknown) {
        this.testResult = 'fail';
        this.testError = e instanceof Error ? e.message : String(e);
      } finally {
        this.testLoading = false;
      }
    },
  },
};
</script>
