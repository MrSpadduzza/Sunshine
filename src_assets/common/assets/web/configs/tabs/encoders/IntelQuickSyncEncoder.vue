<script setup>
import { ref } from 'vue'
import Checkbox from "../../../Checkbox.vue";

const props = defineProps([
  'platform',
  'config',
])

const config = ref(props.config)
</script>

<template>
  <div id="intel-quicksync-encoder" class="config-page">
    <!-- QuickSync Preset -->
    <div class="mb-3">
      <label for="qsv_preset" class="form-label">{{ $t('config.qsv_preset') }}</label>
      <select id="qsv_preset" class="form-select" v-model="config.qsv_preset">
        <option value="veryfast">{{ $t('config.qsv_preset_veryfast') }}</option>
        <option value="faster">{{ $t('config.qsv_preset_faster') }}</option>
        <option value="fast">{{ $t('config.qsv_preset_fast') }}</option>
        <option value="medium">{{ $t('config.qsv_preset_medium') }}</option>
        <option value="slow">{{ $t('config.qsv_preset_slow') }}</option>
        <option value="slower">{{ $t('config.qsv_preset_slower') }}</option>
        <option value="slowest">{{ $t('config.qsv_preset_slowest') }}</option>
      </select>
    </div>

    <!-- QuickSync Coder (H264) -->
    <div class="mb-3">
      <label for="qsv_coder" class="form-label">{{ $t('config.qsv_coder') }}</label>
      <select id="qsv_coder" class="form-select" v-model="config.qsv_coder">
        <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
        <option value="cabac">{{ $t('config.coder_cabac') }}</option>
        <option value="cavlc">{{ $t('config.coder_cavlc') }}</option>
      </select>
    </div>

    <!-- Allow Slow HEVC Encoding -->
    <Checkbox class="mb-3"
              id="qsv_slow_hevc"
              locale-prefix="config"
              v-model="config.qsv_slow_hevc"
              default="false"
    ></Checkbox>

    <!-- Accordion for advanced QSV encoder settings -->
    <div class="mb-3 accordion">
      <div class="accordion-item">
        <h2 class="accordion-header">
          <button class="accordion-button" type="button" data-bs-toggle="collapse"
                  data-bs-target="#qsv-advanced-options">
            {{ $t('config.qsv_advanced') }}
          </button>
        </h2>

        <div id="qsv-advanced-options" class="accordion-collapse collapse">
          <div class="accordion-body">

            <div class="mb-3">
              <label for="qsv_async_depth" class="form-label">{{ $t('config.qsv_async_depth') }}</label>
              <input type="number" min="1" max="16" class="form-control" id="qsv_async_depth"
                    placeholder="1" v-model="config.qsv_async_depth" />
              <div class="form-text">{{ $t('config.qsv_async_depth_desc') }}</div>
            </div>

            <div class="mb-3">
              <label for="qsv_low_power" class="form-label">{{ $t('config.qsv_low_power') }}</label>
              <select id="qsv_low_power" class="form-select" v-model="config.qsv_low_power">
                <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                <option value="enabled">{{ $t('_common.enabled') }}</option>
                <option value="disabled">{{ $t('_common.disabled') }}</option>
              </select>
            </div>

            <div class="mb-3">
              <label for="qsv_low_delay_brc" class="form-label">{{ $t('config.qsv_low_delay_brc') }}</label>
              <select id="qsv_low_delay_brc" class="form-select" v-model="config.qsv_low_delay_brc">
                <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                <option value="enabled">{{ $t('_common.enabled') }}</option>
                <option value="disabled">{{ $t('_common.disabled') }}</option>
              </select>
            </div>

            <div class="mb-3">
              <label for="qsv_extbrc" class="form-label">{{ $t('config.qsv_extbrc') }}</label>
              <select id="qsv_extbrc" class="form-select" v-model="config.qsv_extbrc">
                <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                <option value="enabled">{{ $t('_common.enabled') }}</option>
                <option value="disabled">{{ $t('_common.disabled') }}</option>
              </select>
            </div>

            <div class="mb-3">
              <label for="qsv_look_ahead" class="form-label">{{ $t('config.qsv_look_ahead') }}</label>
              <select id="qsv_look_ahead" class="form-select" v-model="config.qsv_look_ahead">
                <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                <option value="enabled">{{ $t('_common.enabled') }}</option>
                <option value="disabled">{{ $t('_common.disabled') }}</option>
              </select>
            </div>

            <div class="mb-3">
              <label for="qsv_look_ahead_depth" class="form-label">{{ $t('config.qsv_look_ahead_depth') }}</label>
              <input type="number" min="0" max="100" class="form-control" id="qsv_look_ahead_depth"
                    placeholder="0" v-model="config.qsv_look_ahead_depth" />
            </div>

            <div class="mb-3">
              <label for="qsv_mbbrc" class="form-label">{{ $t('config.qsv_mbbrc') }}</label>
              <select id="qsv_mbbrc" class="form-select" v-model="config.qsv_mbbrc">
                <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                <option value="enabled">{{ $t('_common.enabled') }}</option>
                <option value="disabled">{{ $t('_common.disabled') }}</option>
              </select>
            </div>

            <div class="mb-3">
              <label for="qsv_adaptive_i" class="form-label">{{ $t('config.qsv_adaptive_i') }}</label>
              <select id="qsv_adaptive_i" class="form-select" v-model="config.qsv_adaptive_i">
                <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                <option value="enabled">{{ $t('_common.enabled') }}</option>
                <option value="disabled">{{ $t('_common.disabled') }}</option>
              </select>
            </div>

            <div class="mb-3">
              <label for="qsv_adaptive_b" class="form-label">{{ $t('config.qsv_adaptive_b') }}</label>
              <select id="qsv_adaptive_b" class="form-select" v-model="config.qsv_adaptive_b">
                <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                <option value="enabled">{{ $t('_common.enabled') }}</option>
                <option value="disabled">{{ $t('_common.disabled') }}</option>
              </select>
            </div>

          </div>
        </div>
      </div>
    </div>    
  </div>
</template>

<style scoped>

</style>
