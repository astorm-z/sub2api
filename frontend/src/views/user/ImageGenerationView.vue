<template>
  <AppLayout>
    <div class="mx-auto max-w-7xl space-y-6">
      <div class="flex flex-col gap-2">
        <div class="flex items-center gap-3">
          <div class="flex h-11 w-11 items-center justify-center rounded-2xl bg-primary-100 text-primary-600 dark:bg-primary-900/30 dark:text-primary-300">
            <Icon name="sparkles" size="lg" />
          </div>
          <div>
            <h1 class="text-2xl font-semibold text-gray-900 dark:text-white">
              {{ t('imageGeneration.title') }}
            </h1>
            <p class="text-sm text-gray-500 dark:text-gray-400">
              {{ t('imageGeneration.description') }}
            </p>
          </div>
        </div>
        <div class="flex flex-wrap items-center gap-2 text-xs text-gray-500 dark:text-gray-400">
          <span class="rounded-full bg-gray-100 px-2.5 py-1 dark:bg-dark-800">
            {{ t(endpointTextKey) }}
          </span>
          <span class="rounded-full bg-gray-100 px-2.5 py-1 dark:bg-dark-800">
            {{ selectedKeySummary }}
          </span>
          <span
            v-if="lastSavedAtText"
            class="rounded-full bg-primary-50 px-2.5 py-1 text-primary-700 dark:bg-primary-900/20 dark:text-primary-300"
          >
            {{ t('imageGeneration.lastCachedAt', { time: lastSavedAtText }) }}
          </span>
        </div>
      </div>

      <div class="grid grid-cols-1 gap-6 xl:grid-cols-[minmax(0,420px)_minmax(0,1fr)]">
        <section class="card space-y-5 p-5">
          <div class="space-y-1.5">
            <label class="input-label">{{ t('imageGeneration.form.apiKey') }}</label>
            <Select
              v-model="selectedKeyId"
              :options="keyOptions"
              :disabled="loadingKeys || submitting || eligibleKeys.length === 0"
              searchable
              :placeholder="loadingKeys ? t('common.loading') + '...' : t('imageGeneration.form.apiKeyPlaceholder')"
            >
              <template #selected="{ option }">
                <span v-if="option">
                  {{ String(option.label) }}
                </span>
                <span v-else class="text-gray-400">
                  {{ t('imageGeneration.form.apiKeyPlaceholder') }}
                </span>
              </template>
            </Select>
            <p
              v-if="eligibleKeys.length === 0 && !loadingKeys"
              class="rounded-xl border border-amber-200 bg-amber-50 px-3 py-2 text-sm text-amber-700 dark:border-amber-500/20 dark:bg-amber-500/10 dark:text-amber-300"
            >
              {{ t('imageGeneration.noEligibleKeys') }}
            </p>
          </div>

          <div class="grid grid-cols-1 gap-4 sm:grid-cols-2">
            <div class="space-y-1.5">
              <label class="input-label">{{ t('imageGeneration.form.mode') }}</label>
              <Select
                v-model="mode"
                :options="modeOptions"
                :disabled="submitting"
              />
            </div>

            <div class="space-y-1.5">
              <label class="input-label">{{ t('imageGeneration.form.model') }}</label>
              <Input
                v-model="model"
                :disabled="submitting"
                :placeholder="t('imageGeneration.form.modelPlaceholder')"
              />
            </div>
          </div>

          <TextArea
            v-model="prompt"
            :label="t('imageGeneration.form.prompt')"
            :placeholder="t(promptPlaceholderKey)"
            :hint="mode === 'generate' ? t('imageGeneration.form.promptHint') : t('imageGeneration.form.editPromptHint')"
            :disabled="submitting"
            rows="4"
          />

          <div
            v-if="mode === 'edit'"
            class="space-y-4 rounded-2xl border border-gray-200 bg-gray-50 p-4 dark:border-dark-700 dark:bg-dark-900/40"
          >
            <div class="space-y-2">
              <div class="flex items-center justify-between gap-3">
                <div>
                  <div class="text-sm font-medium text-gray-900 dark:text-white">
                    {{ t('imageGeneration.form.sourceImages') }}
                  </div>
                  <p class="text-xs text-gray-500 dark:text-gray-400">
                    {{ t('imageGeneration.form.sourceImagesHint') }}
                  </p>
                </div>
                <label class="btn btn-secondary btn-sm cursor-pointer">
                  <input
                    type="file"
                    accept="image/*"
                    multiple
                    class="hidden"
                    :disabled="submitting"
                    @change="handleSourceImagesChange"
                  />
                  <Icon name="plus" size="sm" class="mr-1.5" />
                  {{ t('imageGeneration.form.addImages') }}
                </label>
              </div>

              <div
                v-if="sourceImagePreviews.length > 0"
                class="grid grid-cols-2 gap-3 sm:grid-cols-3"
              >
                <div
                  v-for="preview in sourceImagePreviews"
                  :key="preview.id"
                  class="group relative overflow-hidden rounded-2xl border border-gray-200 bg-white dark:border-dark-600 dark:bg-dark-800"
                >
                  <img
                    :src="preview.url"
                    :alt="preview.file.name"
                    class="h-28 w-full object-cover"
                  />
                  <div class="flex items-center justify-between gap-2 border-t border-gray-100 px-3 py-2 text-xs dark:border-dark-600">
                    <span class="truncate text-gray-600 dark:text-gray-300">{{ preview.file.name }}</span>
                    <button
                      type="button"
                      class="text-red-500 transition hover:text-red-600"
                      :title="t('common.remove')"
                      :disabled="submitting"
                      @click="removeSourceImage(preview.id)"
                    >
                      <Icon name="trash" size="sm" />
                    </button>
                  </div>
                </div>
              </div>

              <div
                v-else
                class="rounded-2xl border border-dashed border-gray-300 px-4 py-6 text-center text-sm text-gray-500 dark:border-dark-600 dark:text-gray-400"
              >
                {{ t('imageGeneration.form.noSourceImages') }}
              </div>
            </div>

            <div class="space-y-2">
              <div class="flex items-center justify-between gap-3">
                <div>
                  <div class="text-sm font-medium text-gray-900 dark:text-white">
                    {{ t('imageGeneration.form.maskImage') }}
                  </div>
                  <p class="text-xs text-gray-500 dark:text-gray-400">
                    {{ t('imageGeneration.form.maskImageHint') }}
                  </p>
                </div>
                <div class="flex items-center gap-2">
                  <label class="btn btn-secondary btn-sm cursor-pointer">
                    <input
                      type="file"
                      accept="image/*"
                      class="hidden"
                      :disabled="submitting"
                      @change="handleMaskImageChange"
                    />
                    <Icon name="plus" size="sm" class="mr-1.5" />
                    {{ maskImagePreview ? t('imageGeneration.form.replaceMask') : t('imageGeneration.form.uploadMask') }}
                  </label>
                  <button
                    v-if="maskImagePreview"
                    type="button"
                    class="btn btn-secondary btn-sm text-red-600 dark:text-red-400"
                    :disabled="submitting"
                    @click="clearMaskImage"
                  >
                    <Icon name="trash" size="sm" class="mr-1.5" />
                    {{ t('common.remove') }}
                  </button>
                </div>
              </div>

              <div
                v-if="maskImagePreview"
                class="overflow-hidden rounded-2xl border border-gray-200 bg-white dark:border-dark-600 dark:bg-dark-800"
              >
                <img
                  :src="maskImagePreview.url"
                  :alt="maskImagePreview.file.name"
                  class="max-h-56 w-full object-contain"
                />
                <div class="border-t border-gray-100 px-3 py-2 text-xs text-gray-600 dark:border-dark-600 dark:text-gray-300">
                  {{ maskImagePreview.file.name }}
                </div>
              </div>
            </div>
          </div>

          <div class="grid grid-cols-1 gap-4 sm:grid-cols-2">
            <div class="space-y-1.5">
              <label class="input-label">{{ t('imageGeneration.form.size') }}</label>
              <Select
                v-model="size"
                :options="sizeOptions"
                :disabled="submitting"
              />
            </div>

            <div class="space-y-1.5">
              <label class="input-label">{{ t('imageGeneration.form.count') }}</label>
              <Input
                v-model="n"
                type="number"
                :disabled="submitting"
                placeholder="1"
              />
            </div>
          </div>

          <div class="rounded-2xl border border-gray-200 bg-white dark:border-dark-700 dark:bg-dark-900/30">
            <button
              type="button"
              class="flex w-full items-center justify-between gap-3 px-4 py-3 text-left"
              @click="advancedOpen = !advancedOpen"
            >
              <div>
                <div class="text-sm font-medium text-gray-900 dark:text-white">
                  {{ t('imageGeneration.form.advanced') }}
                </div>
                <p class="text-xs text-gray-500 dark:text-gray-400">
                  {{ t('imageGeneration.form.advancedHint') }}
                </p>
              </div>
              <Icon
                name="chevronDown"
                size="sm"
                class="text-gray-400 transition-transform"
                :class="advancedOpen ? 'rotate-180' : ''"
              />
            </button>

            <div
              v-if="advancedOpen"
              class="grid grid-cols-1 gap-4 border-t border-gray-100 px-4 py-4 sm:grid-cols-2 dark:border-dark-700"
            >
              <div class="space-y-1.5">
                <label class="input-label">{{ t('imageGeneration.form.quality') }}</label>
                <Select
                  v-model="quality"
                  :options="qualityOptions"
                  :disabled="submitting"
                />
              </div>

              <div class="space-y-1.5">
                <label class="input-label">{{ t('imageGeneration.form.background') }}</label>
                <Select
                  v-model="background"
                  :options="backgroundOptions"
                  :disabled="submitting"
                />
              </div>

              <div class="space-y-1.5">
                <label class="input-label">{{ t('imageGeneration.form.outputFormat') }}</label>
                <Select
                  v-model="outputFormat"
                  :options="outputFormatOptions"
                  :disabled="submitting"
                />
              </div>

              <div class="space-y-1.5">
                <label class="input-label">{{ t('imageGeneration.form.style') }}</label>
                <Select
                  v-model="style"
                  :options="styleOptions"
                  :disabled="submitting"
                />
              </div>

              <div class="space-y-1.5">
                <div class="mb-1.5 flex items-center gap-1">
                  <label class="input-label mb-0">{{ t('imageGeneration.form.moderation') }}</label>
                  <HelpTooltip :content="t('imageGeneration.form.moderationHelp')">
                    <template #trigger>
                      <Icon
                        name="questionCircle"
                        size="sm"
                        :stroke-width="2"
                        class="cursor-help text-gray-400 transition-colors hover:text-primary-500 dark:text-gray-500 dark:hover:text-primary-400"
                      />
                    </template>
                  </HelpTooltip>
                </div>
                <Select
                  v-model="moderation"
                  :options="moderationOptions"
                  :disabled="submitting"
                />
              </div>

              <div class="space-y-1.5">
                <div class="mb-1.5 flex items-center gap-1">
                  <label class="input-label mb-0">{{ t('imageGeneration.form.outputCompression') }}</label>
                  <HelpTooltip :content="t('imageGeneration.form.outputCompressionHelp')">
                    <template #trigger>
                      <Icon
                        name="questionCircle"
                        size="sm"
                        :stroke-width="2"
                        class="cursor-help text-gray-400 transition-colors hover:text-primary-500 dark:text-gray-500 dark:hover:text-primary-400"
                      />
                    </template>
                  </HelpTooltip>
                </div>
                <Input
                  v-model="outputCompression"
                  type="number"
                  :placeholder="t('imageGeneration.form.outputCompressionPlaceholder')"
                  :disabled="submitting"
                />
              </div>

              <div class="space-y-1.5">
                <div class="mb-1.5 flex items-center gap-1">
                  <label class="input-label mb-0">{{ t('imageGeneration.form.partialImages') }}</label>
                  <HelpTooltip :content="t('imageGeneration.form.partialImagesHelp')">
                    <template #trigger>
                      <Icon
                        name="questionCircle"
                        size="sm"
                        :stroke-width="2"
                        class="cursor-help text-gray-400 transition-colors hover:text-primary-500 dark:text-gray-500 dark:hover:text-primary-400"
                      />
                    </template>
                  </HelpTooltip>
                </div>
                <Input
                  v-model="partialImages"
                  type="number"
                  :placeholder="t('imageGeneration.form.partialImagesPlaceholder')"
                  :disabled="submitting"
                />
              </div>
            </div>
          </div>

          <div class="flex flex-wrap items-center justify-end gap-3">
            <button
              type="button"
              class="btn btn-secondary"
              :disabled="submitting || !hasDirtyResult"
              @click="clearCachedResult"
            >
              <Icon name="trash" size="sm" class="mr-1.5" />
              {{ t('imageGeneration.actions.clearCache') }}
            </button>
            <button
              type="button"
              class="btn btn-secondary"
              :disabled="submitting"
              @click="resetForm"
            >
              <Icon name="refresh" size="sm" class="mr-1.5" />
              {{ t('imageGeneration.actions.resetForm') }}
            </button>
            <button
              type="button"
              :class="submitting ? 'btn btn-secondary' : 'btn btn-primary'"
              :disabled="submitDisabled"
              @click="submitting ? cancelGeneration() : submitGeneration()"
            >
              <Icon
                :name="submitting ? 'x' : 'sparkles'"
                size="sm"
                class="mr-1.5"
              />
              {{ submitting ? t('imageGeneration.actions.cancel') : t('imageGeneration.actions.generate') }}
            </button>
          </div>
        </section>

        <section class="card min-h-[480px] space-y-5 p-5">
          <div class="flex flex-wrap items-center justify-between gap-3">
            <div>
              <h2 class="text-lg font-semibold text-gray-900 dark:text-white">
                {{ t('imageGeneration.result.title') }}
              </h2>
              <p class="text-sm text-gray-500 dark:text-gray-400">
                {{ resultSummaryText }}
              </p>
            </div>

            <div
              v-if="resultImages.length > 0"
              class="flex flex-wrap items-center gap-2"
            >
              <button
                type="button"
                class="btn btn-secondary btn-sm"
                @click="copyAllImageLinks"
              >
                <Icon name="link" size="sm" class="mr-1.5" />
                {{ t('imageGeneration.result.copyAll') }}
              </button>
              <button
                type="button"
                class="btn btn-secondary btn-sm"
                @click="downloadAllImages"
              >
                <Icon name="link" size="sm" class="mr-1.5" />
                {{ t('imageGeneration.result.downloadAll') }}
              </button>
            </div>
          </div>

          <div
            v-if="submitting"
            class="flex min-h-[360px] flex-col items-center justify-center gap-4 rounded-2xl border border-dashed border-primary-200 bg-primary-50/60 px-6 py-10 text-center dark:border-primary-800 dark:bg-primary-900/10"
          >
            <div class="flex h-14 w-14 items-center justify-center rounded-full bg-primary-100 text-primary-600 dark:bg-primary-900/30 dark:text-primary-300">
              <Icon name="refresh" size="lg" class="animate-spin" />
            </div>
            <div>
              <div class="text-base font-medium text-gray-900 dark:text-white">
                {{ t('imageGeneration.result.generating') }}
              </div>
              <p class="mt-1 text-sm text-gray-500 dark:text-gray-400">
                {{ t('imageGeneration.result.generatingHint') }}
              </p>
            </div>
          </div>

          <div
            v-else-if="submitError"
            class="flex min-h-[360px] flex-col items-center justify-center gap-4 rounded-2xl border border-red-200 bg-red-50 px-6 py-10 text-center dark:border-red-500/20 dark:bg-red-500/10"
          >
            <div class="flex h-14 w-14 items-center justify-center rounded-full bg-red-100 text-red-500 dark:bg-red-500/20 dark:text-red-300">
              <Icon name="x" size="lg" />
            </div>
            <div class="space-y-1">
              <div class="text-base font-medium text-red-700 dark:text-red-300">
                {{ t('imageGeneration.result.failed') }}
              </div>
              <p class="max-w-xl text-sm text-red-600/90 dark:text-red-200">
                {{ submitError }}
              </p>
            </div>
          </div>

          <div
            v-else-if="resultImages.length === 0"
            class="flex min-h-[360px] flex-col items-center justify-center gap-4 rounded-2xl border border-dashed border-gray-300 bg-gray-50 px-6 py-10 text-center dark:border-dark-600 dark:bg-dark-900/30"
          >
            <div class="flex h-14 w-14 items-center justify-center rounded-full bg-gray-100 text-gray-500 dark:bg-dark-800 dark:text-gray-300">
              <Icon name="sparkles" size="lg" />
            </div>
            <div>
              <div class="text-base font-medium text-gray-900 dark:text-white">
                {{ t('imageGeneration.result.emptyTitle') }}
              </div>
              <p class="mt-1 text-sm text-gray-500 dark:text-gray-400">
                {{ t('imageGeneration.result.emptyDescription') }}
              </p>
            </div>
          </div>

          <div v-else class="space-y-4">
            <div
              class="grid grid-cols-1 gap-4 lg:grid-cols-2"
            >
              <article
                v-for="(image, index) in resultImages"
                :key="`${image.url}-${index}`"
                class="overflow-hidden rounded-2xl border border-gray-200 bg-white shadow-sm dark:border-dark-600 dark:bg-dark-800"
              >
                <button
                  type="button"
                  class="block w-full bg-gray-50 dark:bg-dark-900/20"
                  @click="previewImageUrl = image.url"
                >
                  <img
                    :src="image.url"
                    :alt="`generated-image-${index + 1}`"
                    class="max-h-[420px] w-full object-contain"
                  />
                </button>

                <div class="space-y-3 border-t border-gray-100 p-4 dark:border-dark-700">
                  <div class="flex items-center justify-between gap-3">
                    <div class="min-w-0">
                      <div class="truncate text-sm font-medium text-gray-900 dark:text-white">
                        {{ image.fileName }}
                      </div>
                      <div class="text-xs text-gray-500 dark:text-gray-400">
                        {{ image.mimeType }}
                      </div>
                    </div>
                    <span class="rounded-full bg-gray-100 px-2 py-1 text-xs text-gray-600 dark:bg-dark-700 dark:text-gray-300">
                      #{{ index + 1 }}
                    </span>
                  </div>

                  <p
                    v-if="image.revisedPrompt"
                    class="rounded-xl bg-gray-50 px-3 py-2 text-xs leading-5 text-gray-600 dark:bg-dark-900/40 dark:text-gray-300"
                  >
                    {{ image.revisedPrompt }}
                  </p>

                  <div class="flex flex-wrap items-center gap-2">
                    <button
                      type="button"
                      class="btn btn-secondary btn-sm"
                      @click="downloadImage(image, index)"
                    >
                      <Icon name="link" size="sm" class="mr-1.5" />
                      {{ t('imageGeneration.result.download') }}
                    </button>
                    <button
                      type="button"
                      class="btn btn-secondary btn-sm"
                      @click="copyImageLink(image)"
                    >
                      <Icon name="link" size="sm" class="mr-1.5" />
                      {{ t('imageGeneration.result.copyLink') }}
                    </button>
                    <button
                      type="button"
                      class="btn btn-secondary btn-sm"
                      @click="previewImageUrl = image.url"
                    >
                      <Icon name="eye" size="sm" class="mr-1.5" />
                      {{ t('imageGeneration.result.preview') }}
                    </button>
                  </div>
                </div>
              </article>
            </div>
          </div>
        </section>
      </div>

      <Teleport to="body">
        <Transition name="fade">
          <div
            v-if="previewImageUrl"
            class="fixed inset-0 z-[100] flex items-center justify-center bg-black/80 p-4"
            @click.self="previewImageUrl = ''"
          >
            <button
              type="button"
              class="absolute right-4 top-4 rounded-full bg-black/50 p-2 text-white transition-colors hover:bg-black/70"
              @click="previewImageUrl = ''"
            >
              <Icon name="x" size="lg" />
            </button>
            <img
              :src="previewImageUrl"
              alt="preview"
              class="max-h-[90vh] max-w-[90vw] rounded-lg object-contain shadow-2xl"
            />
          </div>
        </Transition>
      </Teleport>
    </div>
  </AppLayout>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { keysAPI } from '@/api/keys'
import type { ApiKey } from '@/types'
import AppLayout from '@/components/layout/AppLayout.vue'
import HelpTooltip from '@/components/common/HelpTooltip.vue'
import Select from '@/components/common/Select.vue'
import Input from '@/components/common/Input.vue'
import TextArea from '@/components/common/TextArea.vue'
import { Icon } from '@/components/icons'
import { useAppStore } from '@/stores/app'
import { extractApiErrorMessage } from '@/utils/apiError'

type ImageMode = 'generate' | 'edit'

interface LocalImagePreview {
  id: string
  file: File
  url: string
}

interface ResultImage {
  url: string
  mimeType: string
  fileName: string
  revisedPrompt?: string
}

interface CachedImageGenerationResult {
  version: 1
  savedAt: string
  selectedKeyId: number | null
  form: {
    mode: ImageMode
    prompt: string
    model: string
    size: string
    n: string
    quality: string
    background: string
    outputFormat: string
    moderation: string
    style: string
    outputCompression: string
    partialImages: string
    advancedOpen: boolean
  }
  results: ResultImage[]
}

interface ImagesApiResponseDataItem {
  b64_json?: string
  url?: string
  revised_prompt?: string
}

interface ImagesApiResponse {
  created?: number
  data?: ImagesApiResponseDataItem[]
  output_format?: string
}

const LOCAL_CACHE_KEY = 'sub2api:last-image-generation-result'
const FIXED_RESPONSE_FORMAT = 'url'

const { t, locale } = useI18n()
const appStore = useAppStore()

const loadingKeys = ref(false)
const submitting = ref(false)
const submitError = ref('')
const eligibleKeys = ref<ApiKey[]>([])
const resultImages = ref<ResultImage[]>([])
const previewImageUrl = ref('')
const selectedKeyId = ref<number | null>(null)
const mode = ref<ImageMode>('generate')
const prompt = ref('')
const model = ref('gpt-image-2')
const size = ref('1024x1024')
const n = ref('1')
const quality = ref('')
const background = ref('')
const outputFormat = ref('')
const moderation = ref('')
const style = ref('')
const outputCompression = ref('')
const partialImages = ref('')
const advancedOpen = ref(false)
const sourceImagePreviews = ref<LocalImagePreview[]>([])
const maskImagePreview = ref<LocalImagePreview | null>(null)
const restoredFromCache = ref(false)
const lastSavedAt = ref('')

let abortController: AbortController | null = null

const modeOptions = computed(() => [
  { value: 'generate', label: t('imageGeneration.form.modeGenerate') },
  { value: 'edit', label: t('imageGeneration.form.modeEdit') },
])

const sizeOptions = computed(() => [
  { value: '1024x1024', label: '1024 × 1024' },
  { value: '1536x1024', label: '1536 × 1024' },
  { value: '1024x1536', label: '1024 × 1536' },
  { value: 'auto', label: t('imageGeneration.form.sizeAuto') },
])

const qualityOptions = computed(() => [
  { value: '', label: t('imageGeneration.form.defaultOption') },
  { value: 'low', label: t('imageGeneration.form.qualityLow') },
  { value: 'medium', label: t('imageGeneration.form.qualityMedium') },
  { value: 'high', label: t('imageGeneration.form.qualityHigh') },
  { value: 'auto', label: t('imageGeneration.form.optionAuto') },
])

const backgroundOptions = computed(() => [
  { value: '', label: t('imageGeneration.form.defaultOption') },
  { value: 'transparent', label: t('imageGeneration.form.backgroundTransparent') },
  { value: 'opaque', label: t('imageGeneration.form.backgroundOpaque') },
  { value: 'auto', label: t('imageGeneration.form.optionAuto') },
])

const outputFormatOptions = computed(() => [
  { value: '', label: t('imageGeneration.form.defaultOption') },
  { value: 'png', label: t('imageGeneration.form.outputFormatPng') },
  { value: 'jpeg', label: t('imageGeneration.form.outputFormatJpeg') },
  { value: 'webp', label: t('imageGeneration.form.outputFormatWebp') },
])

const styleOptions = computed(() => [
  { value: '', label: t('imageGeneration.form.defaultOption') },
  { value: 'vivid', label: t('imageGeneration.form.styleVivid') },
  { value: 'natural', label: t('imageGeneration.form.styleNatural') },
])

const moderationOptions = computed(() => [
  { value: '', label: t('imageGeneration.form.defaultOption') },
  { value: 'auto', label: t('imageGeneration.form.optionAuto') },
  { value: 'low', label: t('imageGeneration.form.moderationLow') },
])

const keyOptions = computed(() =>
  eligibleKeys.value.map((key) => ({
    value: key.id,
    label: key.group
      ? `${key.name} · ${key.group.name}`
      : key.name,
  })),
)

const selectedKey = computed(() =>
  eligibleKeys.value.find((key) => key.id === selectedKeyId.value) ?? null,
)

const submitDisabled = computed(() => {
  if (!selectedKey.value || loadingKeys.value) return true
  if (!prompt.value.trim()) return true
  if (submitting.value) return false
  return mode.value === 'edit' && sourceImagePreviews.value.length === 0
})

const promptPlaceholderKey = computed(() =>
  mode.value === 'generate'
    ? 'imageGeneration.form.promptPlaceholder'
    : 'imageGeneration.form.editPromptPlaceholder',
)

const endpointTextKey = computed(() =>
  mode.value === 'generate'
    ? 'imageGeneration.endpointGenerate'
    : 'imageGeneration.endpointEdit',
)

const selectedKeySummary = computed(() => {
  if (loadingKeys.value) {
    return t('imageGeneration.loadingKeys')
  }
  if (!selectedKey.value) {
    return t('imageGeneration.noKeySelected')
  }
  const groupName = selectedKey.value.group?.name
  return groupName
    ? t('imageGeneration.selectedKeySummary', { name: selectedKey.value.name, group: groupName })
    : selectedKey.value.name
})

const lastSavedAtText = computed(() => {
  if (!lastSavedAt.value) return ''
  try {
    return new Date(lastSavedAt.value).toLocaleString(locale.value)
  } catch {
    return lastSavedAt.value
  }
})

const hasDirtyResult = computed(() => resultImages.value.length > 0 || restoredFromCache.value)

const resultSummaryText = computed(() => {
  if (submitting.value) {
    return t('imageGeneration.result.pendingSummary')
  }
  if (resultImages.value.length === 0) {
    return t('imageGeneration.result.emptySummary')
  }
  return restoredFromCache.value
    ? t('imageGeneration.result.cachedSummary', { count: resultImages.value.length })
    : t('imageGeneration.result.readySummary', { count: resultImages.value.length })
})

watch(mode, (value) => {
  submitError.value = ''
  if (value === 'generate') {
    clearLocalPreviews(sourceImagePreviews.value)
    sourceImagePreviews.value = []
    clearMaskImage()
  }
})

watch([selectedKeyId, mode, prompt, model, size, n, quality, background, outputFormat, moderation, style, outputCompression, partialImages, advancedOpen], () => {
  if (!restoredFromCache.value || submitting.value) {
    return
  }
  restoredFromCache.value = false
})

onMounted(async () => {
  restoreCachedResult()
  await loadEligibleKeys()
})

onBeforeUnmount(() => {
  cancelGeneration()
  clearLocalPreviews(sourceImagePreviews.value)
  if (maskImagePreview.value) {
    URL.revokeObjectURL(maskImagePreview.value.url)
  }
})

async function loadEligibleKeys() {
  loadingKeys.value = true
  try {
    const activeKeys: ApiKey[] = []
    let page = 1
    let total = 0
    do {
      const response = await keysAPI.list(page, 100, { status: 'active' })
      activeKeys.push(...response.items)
      total = response.total
      page += 1
    } while (activeKeys.length < total)

    eligibleKeys.value = activeKeys.filter((key) =>
      key.status === 'active' &&
      key.group?.platform === 'openai' &&
      key.group?.status === 'active',
    )

    if (selectedKeyId.value && eligibleKeys.value.some((key) => key.id === selectedKeyId.value)) {
      return
    }

    selectedKeyId.value = eligibleKeys.value[0]?.id ?? null
  } catch (error: unknown) {
    appStore.showError(extractApiErrorMessage(error, t('imageGeneration.errors.loadKeysFailed')))
  } finally {
    loadingKeys.value = false
  }
}

function handleSourceImagesChange(event: Event) {
  const input = event.target as HTMLInputElement
  const files = Array.from(input.files ?? []).filter((file) => file.type.startsWith('image/'))
  if (files.length === 0) {
    input.value = ''
    return
  }

  const nextItems = files.map((file) => ({
    id: `${file.name}-${file.size}-${file.lastModified}-${Math.random().toString(36).slice(2, 8)}`,
    file,
    url: URL.createObjectURL(file),
  }))
  sourceImagePreviews.value = [...sourceImagePreviews.value, ...nextItems]
  input.value = ''
}

function removeSourceImage(id: string) {
  const target = sourceImagePreviews.value.find((item) => item.id === id)
  if (!target) return
  URL.revokeObjectURL(target.url)
  sourceImagePreviews.value = sourceImagePreviews.value.filter((item) => item.id !== id)
}

function handleMaskImageChange(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file || !file.type.startsWith('image/')) {
    input.value = ''
    return
  }
  if (maskImagePreview.value) {
    URL.revokeObjectURL(maskImagePreview.value.url)
  }
  maskImagePreview.value = {
    id: `${file.name}-${file.size}-${file.lastModified}`,
    file,
    url: URL.createObjectURL(file),
  }
  input.value = ''
}

function clearMaskImage() {
  if (maskImagePreview.value) {
    URL.revokeObjectURL(maskImagePreview.value.url)
    maskImagePreview.value = null
  }
}

function clearLocalPreviews(previews: LocalImagePreview[]) {
  for (const preview of previews) {
    URL.revokeObjectURL(preview.url)
  }
}

function resetForm() {
  mode.value = 'generate'
  prompt.value = ''
  model.value = 'gpt-image-2'
  size.value = '1024x1024'
  n.value = '1'
  quality.value = ''
  background.value = ''
  outputFormat.value = ''
  moderation.value = ''
  style.value = ''
  outputCompression.value = ''
  partialImages.value = ''
  advancedOpen.value = false
  submitError.value = ''
  clearLocalPreviews(sourceImagePreviews.value)
  sourceImagePreviews.value = []
  clearMaskImage()
}

async function submitGeneration() {
  if (!selectedKey.value) {
    appStore.showWarning(t('imageGeneration.errors.keyRequired'))
    return
  }
  if (!prompt.value.trim()) {
    appStore.showWarning(t('imageGeneration.errors.promptRequired'))
    return
  }
  if (mode.value === 'edit' && sourceImagePreviews.value.length === 0) {
    appStore.showWarning(t('imageGeneration.errors.sourceImagesRequired'))
    return
  }

  submitError.value = ''
  submitting.value = true
  abortController = new AbortController()

  try {
    const endpoint = mode.value === 'generate'
      ? '/v1/images/generations'
      : '/v1/images/edits'
    const request = buildRequestInit(selectedKey.value.key)
    const response = await fetch(endpoint, {
      ...request,
      signal: abortController.signal,
    })

    const responseText = await response.text()
    if (!response.ok) {
      throw new Error(extractImagesErrorMessage(responseText, response.status))
    }

    const payload = JSON.parse(responseText) as ImagesApiResponse
    const images = normalizeResponseImages(payload)
    if (images.length === 0) {
      throw new Error(t('imageGeneration.errors.noImagesReturned'))
    }

    resultImages.value = images
    restoredFromCache.value = false
    lastSavedAt.value = new Date().toISOString()
    persistCachedResult()
    appStore.showSuccess(t('imageGeneration.messages.generateSuccess', { count: images.length }))
  } catch (error: unknown) {
    if (error instanceof DOMException && error.name === 'AbortError') {
      appStore.showInfo(t('imageGeneration.messages.cancelled'))
      return
    }
    const message = extractApiErrorMessage(error, t('imageGeneration.errors.generateFailed'))
    submitError.value = message
    appStore.showError(message)
  } finally {
    submitting.value = false
    abortController = null
  }
}

function cancelGeneration() {
  if (abortController) {
    abortController.abort()
    abortController = null
  }
  submitting.value = false
}

function buildRequestInit(apiKey: string) {
  const headers: Record<string, string> = {
    Authorization: `Bearer ${apiKey}`,
  }

  if (mode.value === 'generate') {
    headers['Content-Type'] = 'application/json'
    return {
      method: 'POST',
      headers,
      body: JSON.stringify(buildJsonPayload()),
    }
  }

  const formData = buildEditFormData()
  return {
    method: 'POST',
    headers,
    body: formData,
  }
}

function buildJsonPayload() {
  const payload: Record<string, string | number | boolean> = {
    prompt: prompt.value.trim(),
    model: model.value.trim() || 'gpt-image-2',
    response_format: FIXED_RESPONSE_FORMAT,
    stream: false,
  }

  const imageCount = normalizePositiveInt(n.value)
  if (imageCount > 0) payload.n = imageCount
  if (size.value) payload.size = size.value
  if (quality.value) payload.quality = quality.value
  if (background.value) payload.background = background.value
  if (outputFormat.value) payload.output_format = outputFormat.value
  if (moderation.value) payload.moderation = moderation.value
  if (style.value) payload.style = style.value

  const compression = normalizePositiveInt(outputCompression.value)
  if (compression > 0) payload.output_compression = compression

  const partial = normalizePositiveInt(partialImages.value)
  if (partial > 0) payload.partial_images = partial

  return payload
}

function buildEditFormData() {
  const formData = new FormData()
  formData.append('prompt', prompt.value.trim())
  formData.append('model', model.value.trim() || 'gpt-image-2')
  formData.append('response_format', FIXED_RESPONSE_FORMAT)
  formData.append('stream', 'false')

  const imageCount = normalizePositiveInt(n.value)
  if (imageCount > 0) formData.append('n', String(imageCount))
  if (size.value) formData.append('size', size.value)
  if (quality.value) formData.append('quality', quality.value)
  if (background.value) formData.append('background', background.value)
  if (outputFormat.value) formData.append('output_format', outputFormat.value)
  if (moderation.value) formData.append('moderation', moderation.value)
  if (style.value) formData.append('style', style.value)

  const compression = normalizePositiveInt(outputCompression.value)
  if (compression > 0) formData.append('output_compression', String(compression))

  const partial = normalizePositiveInt(partialImages.value)
  if (partial > 0) formData.append('partial_images', String(partial))

  for (const item of sourceImagePreviews.value) {
    formData.append('image', item.file, item.file.name)
  }
  if (maskImagePreview.value) {
    formData.append('mask', maskImagePreview.value.file, maskImagePreview.value.file.name)
  }
  return formData
}

function normalizePositiveInt(value: string): number {
  const parsed = Number.parseInt(value, 10)
  if (!Number.isFinite(parsed) || parsed <= 0) {
    return 0
  }
  return parsed
}

function extractImagesErrorMessage(responseText: string, status: number): string {
  if (!responseText.trim()) {
    return `${t('imageGeneration.errors.generateFailed')} (${status})`
  }

  try {
    const parsed = JSON.parse(responseText) as {
      error?: { message?: string }
      message?: string
      detail?: string
    }
    return parsed.error?.message || parsed.message || parsed.detail || `${t('imageGeneration.errors.generateFailed')} (${status})`
  } catch {
    return responseText
  }
}

function normalizeResponseImages(payload: ImagesApiResponse): ResultImage[] {
  const detectedFormat = payload.output_format || outputFormat.value || 'png'
  const mimeType = resolveMimeType(detectedFormat)
  const items = Array.isArray(payload.data) ? payload.data : []

  return items.reduce<ResultImage[]>((images, item, index) => {
    const dataUrl = normalizeImageUrl(item, mimeType)
    if (!dataUrl) {
      return images
    }
    images.push({
        url: dataUrl,
        mimeType: detectMimeTypeFromUrl(dataUrl, mimeType),
        fileName: buildImageFileName(index, dataUrl, mimeType),
        revisedPrompt: item.revised_prompt?.trim() || undefined,
      })
    return images
  }, [])
}

function normalizeImageUrl(item: ImagesApiResponseDataItem, mimeType: string): string {
  if (item.b64_json) {
    return `data:${mimeType};base64,${item.b64_json}`
  }
  if (item.url) {
    return item.url
  }
  return ''
}

function resolveMimeType(format: string): string {
  const normalized = format.trim().toLowerCase()
  switch (normalized) {
    case 'jpg':
    case 'jpeg':
    case 'image/jpeg':
      return 'image/jpeg'
    case 'webp':
    case 'image/webp':
      return 'image/webp'
    case 'png':
    case 'image/png':
    default:
      return 'image/png'
  }
}

function detectMimeTypeFromUrl(url: string, fallback: string): string {
  if (url.startsWith('data:')) {
    const match = url.match(/^data:([^;]+);/)
    return match?.[1] || fallback
  }
  return fallback
}

function buildImageFileName(index: number, url: string, mimeType: string): string {
  const extension = mimeType.split('/')[1] || 'png'
  if (!url.startsWith('data:')) {
    try {
      const pathname = new URL(url).pathname
      const candidate = pathname.split('/').pop()
      if (candidate) {
        return candidate
      }
    } catch {
      // ignore parse failure
    }
  }
  return `generated-${index + 1}.${extension}`
}

function persistCachedResult() {
  const payload: CachedImageGenerationResult = {
    version: 1,
    savedAt: lastSavedAt.value,
    selectedKeyId: selectedKeyId.value,
    form: {
      mode: mode.value,
      prompt: prompt.value,
      model: model.value,
      size: size.value,
      n: n.value,
      quality: quality.value,
      background: background.value,
      outputFormat: outputFormat.value,
      moderation: moderation.value,
      style: style.value,
      outputCompression: outputCompression.value,
      partialImages: partialImages.value,
      advancedOpen: advancedOpen.value,
    },
    results: resultImages.value,
  }
  localStorage.setItem(LOCAL_CACHE_KEY, JSON.stringify(payload))
}

function restoreCachedResult() {
  const raw = localStorage.getItem(LOCAL_CACHE_KEY)
  if (!raw) {
    return
  }

  try {
    const cached = JSON.parse(raw) as CachedImageGenerationResult
    if (cached.version !== 1) {
      localStorage.removeItem(LOCAL_CACHE_KEY)
      return
    }

    selectedKeyId.value = cached.selectedKeyId
    mode.value = cached.form.mode
    prompt.value = cached.form.prompt
    model.value = cached.form.model
    size.value = cached.form.size
    n.value = cached.form.n
    quality.value = cached.form.quality
    background.value = cached.form.background
    outputFormat.value = cached.form.outputFormat
    moderation.value = cached.form.moderation
    style.value = cached.form.style
    outputCompression.value = cached.form.outputCompression
    partialImages.value = cached.form.partialImages
    advancedOpen.value = cached.form.advancedOpen
    resultImages.value = Array.isArray(cached.results) ? cached.results : []
    restoredFromCache.value = resultImages.value.length > 0
    lastSavedAt.value = cached.savedAt
  } catch {
    localStorage.removeItem(LOCAL_CACHE_KEY)
  }
}

function clearCachedResult() {
  localStorage.removeItem(LOCAL_CACHE_KEY)
  resultImages.value = []
  restoredFromCache.value = false
  submitError.value = ''
  lastSavedAt.value = ''
  appStore.showSuccess(t('imageGeneration.messages.cacheCleared'))
}

async function copyAllImageLinks() {
  const content = resultImages.value.map((image) => image.url).join('\n')
  if (!content) return
  await navigator.clipboard.writeText(content)
  appStore.showSuccess(t('imageGeneration.messages.linksCopied'))
}

async function copyImageLink(image: ResultImage) {
  await navigator.clipboard.writeText(image.url)
  appStore.showSuccess(t('imageGeneration.messages.linkCopied'))
}

async function downloadAllImages() {
  for (const [index, image] of resultImages.value.entries()) {
    await downloadImage(image, index)
  }
}

async function downloadImage(image: ResultImage, index: number) {
  try {
    if (image.url.startsWith('data:')) {
      triggerDownload(image.url, image.fileName)
      return
    }

    const response = await fetch(image.url)
    const blob = await response.blob()
    const objectUrl = URL.createObjectURL(blob)
    triggerDownload(objectUrl, image.fileName || `generated-${index + 1}.png`)
    setTimeout(() => URL.revokeObjectURL(objectUrl), 1000)
  } catch {
    window.open(image.url, '_blank', 'noopener,noreferrer')
    appStore.showInfo(t('imageGeneration.messages.openedInNewTab'))
  }
}

function triggerDownload(url: string, fileName: string) {
  const link = document.createElement('a')
  link.href = url
  link.download = fileName
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
}
</script>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
