<script setup lang="ts">
interface Props {
  length?: number
  n?: number
  error?: boolean
  errorMessage?: string
  disabled?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  length: 6,
  n: 6,
  error: false,
  errorMessage: '',
  disabled: false
})

const model = defineModel<string>({ default: '' })
const otpLength = computed(() => Math.min(Math.max(props.length, 4), 8))
const itemsPerRow = computed(() => Math.min(Math.max(props.n, 1), otpLength.value))
const digits = ref<string[]>(Array(otpLength.value).fill(''))
const inputRefs = ref<HTMLInputElement[]>([])

const currentOtp = computed(() => digits.value.join(''))
const digitRows = computed(() => {
  const rows: number[][] = []

  for (let index = 0; index < otpLength.value; index += itemsPerRow.value) {
    rows.push(
      Array.from(
        { length: Math.min(itemsPerRow.value, otpLength.value - index) },
        (_, offset) => index + offset
      )
    )
  }

  return rows
})

const normalizeOtp = (value: string) => value.replace(/\D/g, '').slice(0, otpLength.value)

const setDigitsFromValue = (value: string) => {
  const normalizedValue = normalizeOtp(value)
  const nextDigits = Array(otpLength.value).fill('')

  normalizedValue.split('').forEach((digit, index) => {
    nextDigits[index] = digit
  })

  digits.value = nextDigits
}

const setInputRef = (el: unknown, index: number) => {
  if (el instanceof HTMLInputElement) {
    inputRefs.value[index] = el
  }
}

const focusInput = async (index: number) => {
  if (props.disabled) {
    return
  }

  await nextTick()
  inputRefs.value[index]?.focus()
  inputRefs.value[index]?.select()
}

const syncModel = () => {
  if (model.value !== currentOtp.value) {
    model.value = currentOtp.value
  }
}

const handleInput = (event: Event, index: number) => {
  const target = event.target as HTMLInputElement
  const value = target.value.replace(/\D/g, '')

  if (!value) {
    digits.value[index] = ''
    target.value = ''
    syncModel()
    return
  }

  digits.value[index] = value.slice(-1)
  target.value = digits.value[index]

  if (index < otpLength.value - 1) {
    focusInput(index + 1)
  }

  syncModel()
}

const handleKeydown = (event: KeyboardEvent, index: number) => {
  if (event.key !== 'Backspace') {
    return
  }

  event.preventDefault()

  if (digits.value[index]) {
    digits.value[index] = ''
    syncModel()
    return
  }

  if (index > 0) {
    digits.value[index - 1] = ''
    focusInput(index - 1)
    syncModel()
  }
}

const handlePaste = (event: ClipboardEvent, index: number) => {
  event.preventDefault()

  const pastedDigits = event.clipboardData?.getData('text').replace(/\D/g, '') ?? ''
  if (!pastedDigits) {
    return
  }

  const startIndex = pastedDigits.length >= otpLength.value ? 0 : index

  pastedDigits
    .slice(0, otpLength.value - startIndex)
    .split('')
    .forEach((digit, offset) => {
      digits.value[startIndex + offset] = digit
    })

  const nextEmptyIndex = digits.value.findIndex((digit, digitIndex) => digitIndex >= startIndex && !digit)
  const focusIndex = nextEmptyIndex === -1 ? otpLength.value - 1 : nextEmptyIndex

  focusInput(focusIndex)
  syncModel()
}

watch(model, (value) => {
  const normalizedValue = normalizeOtp(value)

  if (model.value !== normalizedValue) {
    model.value = normalizedValue
    return
  }

  if (normalizedValue !== currentOtp.value) {
    setDigitsFromValue(normalizedValue)
  }
}, { immediate: true })

watch(otpLength, () => {
  inputRefs.value = []
  setDigitsFromValue(model.value)
  syncModel()
})
</script>

<template>
  <div class="w-full">
    <div
      class="flex flex-col items-center gap-2 sm:gap-3"
      role="group"
      aria-label="OTP 驗證碼"
    >
      <div
        v-for="(row, rowIndex) in digitRows"
        :key="rowIndex"
        class="flex justify-center gap-2 sm:gap-3"
      >
        <input
          v-for="index in row"
          :key="index"
          :ref="(el) => setInputRef(el, index)"
          v-model="digits[index]"
          type="text"
          inputmode="numeric"
          autocomplete="one-time-code"
          maxlength="1"
          :disabled="props.disabled"
          class="h-12 w-10 rounded-lg border-2 bg-white text-center text-xl font-semibold text-slate-950 outline-none transition sm:h-14 sm:w-12 sm:text-2xl"
          :class="[
            props.error
              ? 'border-red-500 focus:border-red-500 focus:ring-2 focus:ring-red-100'
              : 'border-slate-300 focus:border-slate-950 focus:ring-2 focus:ring-slate-200',
            props.disabled ? 'cursor-not-allowed bg-slate-100 text-slate-400' : ''
          ]"
          :aria-invalid="props.error"
          :aria-describedby="props.error && props.errorMessage ? 'otp-error-message' : undefined"
          @input="handleInput($event, index)"
          @keydown="handleKeydown($event, index)"
          @paste="handlePaste($event, index)"
        >
      </div>
    </div>

    <p
      v-if="props.error && props.errorMessage"
      id="otp-error-message"
      class="mt-3 text-center text-sm font-medium text-red-600"
    >
      {{ props.errorMessage }}
    </p>
  </div>
</template>
