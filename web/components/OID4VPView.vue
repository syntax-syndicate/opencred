<!--
Copyright 2023 - 2024 California Department of Motor Vehicles
Copyright 2023 - 2024 Digital Bazaar, Inc.

SPDX-License-Identifier: BSD-3-Clause
-->

<script setup>
import {inject, onMounted, ref} from 'vue';
import CountdownDisplay from './CountdownDisplay.vue';
import {httpClient} from '@digitalbazaar/http-client';
import {useQuasar} from 'quasar';

const props = defineProps({
  active: {
    type: Boolean,
    default: false
  },
  brand: {
    type: Object,
    default: () => ({
      primary: ''
    })
  },
  exchangeData: {
    type: Object,
    default: () => ({
      QR: '',
      vcapi: '',
      OID4VP: '',
      ttl: 900,
      createdAt: new Date(),
      oidc: {
        state: ''
      }
    })
  },
  explainerVideo: {
    type: Object,
    default: () => ({
      id: '',
      provider: ''
    })
  }
});
const emit = defineEmits(['replaceExchange', 'overrideActive']);
const showDeeplink = ref(false);
const showVideo = ref(false);
const showNoSchemeHandlerWarning = ref(false);
const $q = useQuasar();
const $cookies = inject('$cookies');

const switchView = () => {
  showDeeplink.value = !showDeeplink.value;
};

onMounted(() => {
  if($q.platform.is.mobile) {
    showDeeplink.value = true;
  }
});

async function appOpened() {
  const {location} = window;
  const searchParams = new URLSearchParams(location.search);
  const variables = JSON.parse(atob(searchParams.get('variables') || 'e30='));
  const redirectPath = location.href.split(location.origin).at(-1);
  let exchange = {};
  ({
    data: exchange,
  } = await httpClient.post(
    `/workflows/${props.exchangeData.workflowId}` +
    `/exchanges`,
    {
      json: {
        variables: btoa(JSON.stringify({
          ...variables,
          redirectPath
        })).replace(/=+$/, ''),
        oidcState: props.exchangeData.oidc.state
      },
      headers: {
        Authorization: `Bearer ${props.exchangeData.accessToken}`
      }
    }
  ));
  emit('replaceExchange', exchange);
  $cookies.set('exchangeId', exchange.id,
    `${props.exchangeData.ttl}s`,
    '', '', true, 'Strict');
  $cookies.set('accessToken', exchange.accessToken,
    `${props.exchangeData.ttl}s`,
    '', '', true, 'Strict');

  // If we're still on the page after 2 seconds, the app may have failed to
  // launch. Warn the user.
  setTimeout(() => {
    if(window.location !== exchange.OID4VP) {
      showNoSchemeHandlerWarning.value = true;
    }
  }, 2000);
  window.location.replace(exchange.OID4VP);
}

const handleGoBack = () => {
  emit('overrideActive');
};
</script>

<template>
  <div
    class="-mt-72 bg-white z-10 mx-auto p-10 rounded-md max-w-3xl
           px-16 lg:px-24 relative text-center">
    <h1
      class="text-3xl mb-12 text-center"
      :style="{color: brand.primary}">
      {{$t('qrTitle')}}
    </h1>
    <div class="mb-4">
      <p
        v-if="$t('qrPageExplain')"
        v-html="$t('qrPageExplain')" />
      <p
        v-if="$t('qrPageExplainHelp')"
        class="mt-2"
        v-html="$t('qrPageExplainHelp')" />
    </div>
    <div
      v-if="(active && !showDeeplink)"
      class="p-12 m-12 justify-center">
      <!-- Exchange is active: Loading spinner -->
      <div class="mx-auto w-7 mb-4">
        <q-spinner-tail
          color="primary"
          size="2em" />
      </div>
      <p>
        {{$t('exchangeActiveExpiryMessage')}}
        <CountdownDisplay
          :created-at="props.exchangeData.createdAt"
          :ttl="props.exchangeData.ttl" />
      </p>
      <button
        class="mx-auto max-w-prose text-sm underline"
        @click="handleGoBack">
        {{$t('exchangeActiveGoBack')}}
      </button>
    </div>
    <div
      v-else-if="!showDeeplink && exchangeData.QR !== ''"
      class="mb-4 justify-center">
      <!-- Show QR code to scan from a wallet app -->
      <div>
        <img
          v-if="exchangeData.QR !== ''"
          :src="exchangeData.QR"
          class="mx-auto">
      </div>
      <div>
        <p>
          <!-- A button to switch to a deep link button -->
          <span v-if="$t('openid4vpQrAnotherWayLabel')">
            {{$t('openid4vpQrAnotherWayLabel')}}
          </span>
          <button
            class="mt-2 underline pl-1 inline"
            :style="{color: brand.primary}"
            @click="switchView">
            {{$t('openid4vpQrAnotherWay')}}
          </button>
        </p>
      </div>
      <p>
        {{$t('exchangeActiveExpiryMessage')}}
        <CountdownDisplay
          :created-at="props.exchangeData.createdAt"
          :ttl="props.exchangeData.ttl" />
      </p>
    </div>
    <div
      v-else-if="exchangeData.QR"
      class="flex justify-center">
      <!-- A button to launch a same-device wallet -->
      <div v-if="!active">
        <q-btn
          v-if="!active"
          color="primary"
          @click="appOpened()">
          {{$t('appCta')}}
        </q-btn>
        <p
          v-if="showNoSchemeHandlerWarning"
          class="mt-4 text-red-600">
          <span class="text-bold">
            {{$t('noSchemeHandlerTitle')}}
          </span>
          {{$t('noSchemeHandlerMessage')}}
        </p>
      </div>
      <div v-else>
        <q-btn
          color="primary"
          class="px-16 py-4"
          @click="appOpened()">
          <q-spinner-tail
            color="white"
            size="1em" />
        </q-btn>
      </div>
      <div>
        <p class="my-4">
          {{$t('exchangeActiveExpiryMessage')}}
          <CountdownDisplay
            :created-at="props.exchangeData.createdAt"
            :ttl="props.exchangeData.ttl" />
        </p>
      </div>
    </div>
    <div class="mt-2">
      <p v-if="showDeeplink && exchangeData.QR !== ''">
        <!-- A button to switch (back) to a QR code -->
        <span v-if="$t('openid4vpAnotherWayLabel')">
          {{$t('openid4vpAnotherWayLabel')}}
        </span>
        <button
          class="mt-2 underline pl-1 inline"
          :style="{color: brand.primary}"
          @click="switchView">
          {{$t('openid4vpAnotherWay')}}
        </button>
      </p>
      <button
        v-else-if="$t('qrExplainerText') !== ''
          && props.explainerVideo.id !== ''
          && props.explainerVideo.provider"
        :style="{color: brand.primary}"
        class="underline"
        @click="showVideo = true">
        {{$t('qrExplainerText')}}
      </button>
      <p
        v-if="$t('qrFooterHelp')"
        class="mt-2"
        v-html="$t('qrFooterHelp')" />
    </div>
    <div
      v-if="$t('qrDisclaimer')"
      class="mt-12 flex flex-col items-center"
      v-html="$t('qrDisclaimer')" />

    <q-dialog
      v-model="showVideo">
      <q-card>
        <YouTubeVideo
          v-if="explainerVideo.provider === 'youtube'"
          :id="explainerVideo.id" />
        <q-card-actions
          align="right">
          <q-btn
            v-close-popup
            flat
            label="Close" />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </div>
</template>
