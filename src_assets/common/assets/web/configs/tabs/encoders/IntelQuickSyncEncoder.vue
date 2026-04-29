<script setup>
import { computed, ref } from 'vue'
import Checkbox from "../../../Checkbox.vue";

const props = defineProps([
  'platform',
  'config',
])

const config = ref(props.config)

const numberOrNull = (value) => {
  if (value === undefined || value === null || value === '') {
    return null
  }

  const n = Number(value)
  return Number.isFinite(n) ? n : null
}

const triStateEnabled = (value, autoDefault = false) => {
  if (value === undefined || value === null || value === '' || value === 'auto') {
    return autoDefault
  }

  return value === true || value === 1 || value === '1' || value === 'enabled' || value === 'on' || value === 'true'
}

const qsvPredictedRateControl = computed(() => {
  const qscale = triStateEnabled(config.value.qsv_qscale, false)
  const lookAhead = triStateEnabled(config.value.qsv_look_ahead, false)
  const vcm = triStateEnabled(config.value.qsv_vcm, true)
  const globalQuality = numberOrNull(config.value.qsv_global_quality)
  const bitrate = numberOrNull(config.value.qsv_bitrate)
  const maxrate = numberOrNull(config.value.qsv_max_bitrate)
  const avbrAccuracy = numberOrNull(config.value.qsv_avbr_accuracy)
  const avbrConvergence = numberOrNull(config.value.qsv_avbr_convergence)

  const exclusiveModes = [qscale, lookAhead, vcm].filter(Boolean).length
  if (exclusiveModes > 1) {
    return {
      name: 'Invalid / conflicting',
      description: 'FFmpeg QSV accepts only one of fixed qscale, lookahead, or VCM at a time.',
      warning: true,
    }
  }

  if (qscale) {
    return {
      name: 'CQP',
      description: 'Fixed qscale is enabled, so QSV will request constant quantization parameter mode. Set Global Quality to choose the QP value.',
      warning: !(globalQuality && globalQuality > 0),
    }
  }

  if (vcm) {
    return {
      name: 'VCM for H.264; otherwise bitrate/quality-derived',
      description: 'Sunshine historically enables VCM for h264_qsv. Disable QSV VCM to allow H.264 CBR/VBR/QVBR/ICQ selection.',
    }
  }

  if (lookAhead) {
    if (globalQuality && globalQuality > 0) {
      return {
        name: 'LA_ICQ',
        description: 'Lookahead and Global Quality are enabled, so QSV will request intelligent constant quality with lookahead.',
      }
    }

    return {
      name: 'LA',
      description: 'Lookahead is enabled without Global Quality, so QSV will request VBR with lookahead.',
    }
  }

  const maxrateDisabled = maxrate === 0
  const bitrateDisabled = bitrate === 0
  const hasBitrate = !bitrateDisabled
  const hasMaxrate = !maxrateDisabled

  if (globalQuality && globalQuality > 0 && !hasMaxrate) {
    return {
      name: 'ICQ',
      description: 'Global Quality is set and maxrate is disabled, so QSV will request intelligent constant quality.',
    }
  }

  if (hasBitrate) {
    if (hasMaxrate && bitrate !== null && maxrate !== null && bitrate === maxrate) {
      return {
        name: 'CBR',
        description: 'Bitrate and Max Bitrate are both set to the same value.',
      }
    }

    if (!hasMaxrate && avbrAccuracy && avbrAccuracy > 0 && avbrConvergence && avbrConvergence > 0) {
      return {
        name: 'AVBR',
        description: 'Maxrate is disabled and AVBR accuracy/convergence are set. This applies to H.264/HEVC when the QSV runtime supports it.',
      }
    }

    if (globalQuality && globalQuality > 0) {
      return {
        name: 'QVBR',
        description: 'Bitrate is present together with Global Quality, so QSV will request quality-defined VBR.',
      }
    }

    return {
      name: 'VBR',
      description: 'Sunshine supplies a bitrate and maxrate by default; unless they are equal, QSV will request variable bitrate.',
    }
  }

  return {
    name: 'CQP',
    description: 'No bitrate is present, so FFmpeg QSV falls back to constant quantization parameter mode.',
  }
})
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

    <!-- QSV rate-control primitive settings -->
    <div class="mb-3 accordion">
      <div class="accordion-item">
        <h2 class="accordion-header">
          <button class="accordion-button" type="button" data-bs-toggle="collapse"
                  data-bs-target="#qsv-rate-control-options">
            {{ $t('config.qsv_rate_control_options') }}
          </button>
        </h2>

        <div id="qsv-rate-control-options" class="accordion-collapse collapse show">
          <div class="accordion-body">
            <div class="alert mb-3" :class="qsvPredictedRateControl.warning ? 'alert-warning' : 'alert-info'">
              <strong>{{ $t('config.qsv_predicted_rate_control') }}:</strong>
              {{ qsvPredictedRateControl.name }}
              <div class="small mt-1">{{ qsvPredictedRateControl.description }}</div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-6">
                <label for="qsv_bitrate" class="form-label">{{ $t('config.qsv_bitrate') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_bitrate"
                       :placeholder="$t('config.qsv_use_stream_bitrate')" v-model="config.qsv_bitrate" />
                <div class="form-text">{{ $t('config.qsv_bitrate_desc') }}</div>
              </div>

              <div class="mb-3 col-md-6">
                <label for="qsv_max_bitrate" class="form-label">{{ $t('config.qsv_max_bitrate') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_max_bitrate"
                       :placeholder="$t('config.qsv_use_stream_bitrate')" v-model="config.qsv_max_bitrate" />
                <div class="form-text">{{ $t('config.qsv_max_bitrate_desc') }}</div>
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-6">
                <label for="qsv_global_quality" class="form-label">{{ $t('config.qsv_global_quality') }}</label>
                <input type="number" min="0" max="51" class="form-control" id="qsv_global_quality"
                       placeholder="0" v-model="config.qsv_global_quality" />
                <div class="form-text">{{ $t('config.qsv_global_quality_desc') }}</div>
              </div>

              <div class="mb-3 col-md-6">
                <label for="qsv_qscale" class="form-label">{{ $t('config.qsv_qscale') }}</label>
                <select id="qsv_qscale" class="form-select" v-model="config.qsv_qscale">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
                <div class="form-text">{{ $t('config.qsv_qscale_desc') }}</div>
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-6">
                <label for="qsv_rc_buffer_size" class="form-label">{{ $t('config.qsv_rc_buffer_size') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_rc_buffer_size"
                       placeholder="0" v-model="config.qsv_rc_buffer_size" />
              </div>

              <div class="mb-3 col-md-6">
                <label for="qsv_rc_initial_buffer_occupancy" class="form-label">{{ $t('config.qsv_rc_initial_buffer_occupancy') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_rc_initial_buffer_occupancy"
                       placeholder="0" v-model="config.qsv_rc_initial_buffer_occupancy" />
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-4">
                <label for="qsv_vcm" class="form-label">{{ $t('config.qsv_vcm') }}</label>
                <select id="qsv_vcm" class="form-select" v-model="config.qsv_vcm">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
                <div class="form-text">{{ $t('config.qsv_vcm_desc') }}</div>
              </div>

              <div class="mb-3 col-md-4">
                <label for="qsv_avbr_accuracy" class="form-label">{{ $t('config.qsv_avbr_accuracy') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_avbr_accuracy"
                       placeholder="0" v-model="config.qsv_avbr_accuracy" />
              </div>

              <div class="mb-3 col-md-4">
                <label for="qsv_avbr_convergence" class="form-label">{{ $t('config.qsv_avbr_convergence') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_avbr_convergence"
                       placeholder="0" v-model="config.qsv_avbr_convergence" />
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

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

            <div class="row">
              <div class="mb-3 col-md-6">
                <label for="qsv_low_power" class="form-label">{{ $t('config.qsv_low_power') }}</label>
                <select id="qsv_low_power" class="form-select" v-model="config.qsv_low_power">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>

              <div class="mb-3 col-md-6">
                <label for="qsv_low_delay_brc" class="form-label">{{ $t('config.qsv_low_delay_brc') }}</label>
                <select id="qsv_low_delay_brc" class="form-select" v-model="config.qsv_low_delay_brc">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-6">
                <label for="qsv_extbrc" class="form-label">{{ $t('config.qsv_extbrc') }}</label>
                <select id="qsv_extbrc" class="form-select" v-model="config.qsv_extbrc">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>

              <div class="mb-3 col-md-6">
                <label for="qsv_look_ahead" class="form-label">{{ $t('config.qsv_look_ahead') }}</label>
                <select id="qsv_look_ahead" class="form-select" v-model="config.qsv_look_ahead">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-6">
                <label for="qsv_look_ahead_depth" class="form-label">{{ $t('config.qsv_look_ahead_depth') }}</label>
                <input type="number" min="0" max="100" class="form-control" id="qsv_look_ahead_depth"
                       placeholder="0" v-model="config.qsv_look_ahead_depth" />
              </div>

              <div class="mb-3 col-md-6">
                <label for="qsv_look_ahead_downsampling" class="form-label">{{ $t('config.qsv_look_ahead_downsampling') }}</label>
                <select id="qsv_look_ahead_downsampling" class="form-select" v-model="config.qsv_look_ahead_downsampling">
                  <option value="">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="auto">auto</option>
                  <option value="off">off</option>
                  <option value="2x">2x</option>
                  <option value="4x">4x</option>
                </select>
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-4">
                <label for="qsv_mbbrc" class="form-label">{{ $t('config.qsv_mbbrc') }}</label>
                <select id="qsv_mbbrc" class="form-select" v-model="config.qsv_mbbrc">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>

              <div class="mb-3 col-md-4">
                <label for="qsv_adaptive_i" class="form-label">{{ $t('config.qsv_adaptive_i') }}</label>
                <select id="qsv_adaptive_i" class="form-select" v-model="config.qsv_adaptive_i">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>

              <div class="mb-3 col-md-4">
                <label for="qsv_adaptive_b" class="form-label">{{ $t('config.qsv_adaptive_b') }}</label>
                <select id="qsv_adaptive_b" class="form-select" v-model="config.qsv_adaptive_b">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>
            </div>

            <hr />

            <div class="row">
              <div class="mb-3 col-md-4">
                <label for="qsv_max_frame_size" class="form-label">{{ $t('config.qsv_max_frame_size') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_max_frame_size" placeholder="0" v-model="config.qsv_max_frame_size" />
              </div>
              <div class="mb-3 col-md-4">
                <label for="qsv_max_frame_size_i" class="form-label">{{ $t('config.qsv_max_frame_size_i') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_max_frame_size_i" placeholder="0" v-model="config.qsv_max_frame_size_i" />
              </div>
              <div class="mb-3 col-md-4">
                <label for="qsv_max_frame_size_p" class="form-label">{{ $t('config.qsv_max_frame_size_p') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_max_frame_size_p" placeholder="0" v-model="config.qsv_max_frame_size_p" />
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-3">
                <label for="qsv_qmin" class="form-label">{{ $t('config.qsv_qmin') }}</label>
                <input type="number" min="0" max="255" class="form-control" id="qsv_qmin" placeholder="0" v-model="config.qsv_qmin" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_qmax" class="form-label">{{ $t('config.qsv_qmax') }}</label>
                <input type="number" min="0" max="255" class="form-control" id="qsv_qmax" placeholder="0" v-model="config.qsv_qmax" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_min_qp_i" class="form-label">{{ $t('config.qsv_min_qp_i') }}</label>
                <input type="number" min="0" max="255" class="form-control" id="qsv_min_qp_i" placeholder="0" v-model="config.qsv_min_qp_i" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_max_qp_i" class="form-label">{{ $t('config.qsv_max_qp_i') }}</label>
                <input type="number" min="0" max="255" class="form-control" id="qsv_max_qp_i" placeholder="0" v-model="config.qsv_max_qp_i" />
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-3">
                <label for="qsv_min_qp_p" class="form-label">{{ $t('config.qsv_min_qp_p') }}</label>
                <input type="number" min="0" max="255" class="form-control" id="qsv_min_qp_p" placeholder="0" v-model="config.qsv_min_qp_p" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_max_qp_p" class="form-label">{{ $t('config.qsv_max_qp_p') }}</label>
                <input type="number" min="0" max="255" class="form-control" id="qsv_max_qp_p" placeholder="0" v-model="config.qsv_max_qp_p" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_min_qp_b" class="form-label">{{ $t('config.qsv_min_qp_b') }}</label>
                <input type="number" min="0" max="255" class="form-control" id="qsv_min_qp_b" placeholder="0" v-model="config.qsv_min_qp_b" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_max_qp_b" class="form-label">{{ $t('config.qsv_max_qp_b') }}</label>
                <input type="number" min="0" max="255" class="form-control" id="qsv_max_qp_b" placeholder="0" v-model="config.qsv_max_qp_b" />
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-4">
                <label for="qsv_bitrate_limit" class="form-label">{{ $t('config.qsv_bitrate_limit') }}</label>
                <select id="qsv_bitrate_limit" class="form-select" v-model="config.qsv_bitrate_limit">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>
              <div class="mb-3 col-md-4">
                <label for="qsv_rdo" class="form-label">{{ $t('config.qsv_rdo') }}</label>
                <select id="qsv_rdo" class="form-select" v-model="config.qsv_rdo">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>
              <div class="mb-3 col-md-4">
                <label for="qsv_b_strategy" class="form-label">{{ $t('config.qsv_b_strategy') }}</label>
                <select id="qsv_b_strategy" class="form-select" v-model="config.qsv_b_strategy">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-3">
                <label for="qsv_p_strategy" class="form-label">{{ $t('config.qsv_p_strategy') }}</label>
                <input type="number" min="0" max="2" class="form-control" id="qsv_p_strategy" placeholder="0" v-model="config.qsv_p_strategy" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_dblk_idc" class="form-label">{{ $t('config.qsv_dblk_idc') }}</label>
                <input type="number" min="0" max="2" class="form-control" id="qsv_dblk_idc" placeholder="0" v-model="config.qsv_dblk_idc" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_idr_interval" class="form-label">{{ $t('config.qsv_idr_interval') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_idr_interval" placeholder="0" v-model="config.qsv_idr_interval" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_gpb" class="form-label">{{ $t('config.qsv_gpb') }}</label>
                <select id="qsv_gpb" class="form-select" v-model="config.qsv_gpb">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-3">
                <label for="qsv_aud" class="form-label">{{ $t('config.qsv_aud') }}</label>
                <select id="qsv_aud" class="form-select" v-model="config.qsv_aud">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_repeat_pps" class="form-label">{{ $t('config.qsv_repeat_pps') }}</label>
                <select id="qsv_repeat_pps" class="form-select" v-model="config.qsv_repeat_pps">
                  <option value="auto">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="enabled">{{ $t('_common.enabled') }}</option>
                  <option value="disabled">{{ $t('_common.disabled') }}</option>
                </select>
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_tile_cols" class="form-label">{{ $t('config.qsv_tile_cols') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_tile_cols" placeholder="0" v-model="config.qsv_tile_cols" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_tile_rows" class="form-label">{{ $t('config.qsv_tile_rows') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_tile_rows" placeholder="0" v-model="config.qsv_tile_rows" />
              </div>
            </div>

            <div class="row">
              <div class="mb-3 col-md-3">
                <label for="qsv_int_ref_type" class="form-label">{{ $t('config.qsv_int_ref_type') }}</label>
                <select id="qsv_int_ref_type" class="form-select" v-model="config.qsv_int_ref_type">
                  <option value="">{{ $t('config.ffmpeg_auto') }}</option>
                  <option value="none">none</option>
                  <option value="vertical">vertical</option>
                  <option value="horizontal">horizontal</option>
                  <option value="slice">slice</option>
                </select>
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_int_ref_cycle_size" class="form-label">{{ $t('config.qsv_int_ref_cycle_size') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_int_ref_cycle_size" placeholder="0" v-model="config.qsv_int_ref_cycle_size" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_int_ref_qp_delta" class="form-label">{{ $t('config.qsv_int_ref_qp_delta') }}</label>
                <input type="number" min="-75" max="75" class="form-control" id="qsv_int_ref_qp_delta" placeholder="0" v-model="config.qsv_int_ref_qp_delta" />
              </div>
              <div class="mb-3 col-md-3">
                <label for="qsv_int_ref_cycle_dist" class="form-label">{{ $t('config.qsv_int_ref_cycle_dist') }}</label>
                <input type="number" min="0" class="form-control" id="qsv_int_ref_cycle_dist" placeholder="0" v-model="config.qsv_int_ref_cycle_dist" />
              </div>
            </div>

            <div class="mb-3">
              <label for="qsv_scenario" class="form-label">{{ $t('config.qsv_scenario') }}</label>
              <select id="qsv_scenario" class="form-select" v-model="config.qsv_scenario">
                <option value="">{{ $t('config.ffmpeg_auto') }}</option>
                <option value="unknown">unknown</option>
                <option value="displayremoting">displayremoting</option>
                <option value="videoconference">videoconference</option>
                <option value="archive">archive</option>
                <option value="livestreaming">livestreaming</option>
                <option value="cameracapture">cameracapture</option>
                <option value="videosurveillance">videosurveillance</option>
                <option value="gamestreaming">gamestreaming</option>
                <option value="remotegaming">remotegaming</option>
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
