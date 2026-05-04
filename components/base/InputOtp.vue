<script setup lang="ts">
interface Props {
  length?: number
  error?: boolean
  errorMessage?: string
  disabled?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  length: 6,
  error: false,
  errorMessage: '',
  disabled: false
})

const emit = defineEmits<{
  complete: [otp: string]
  change: [otp: string]
}>()

const otpLength = computed(() => Math.min(Math.max(props.length, 4), 8))
const digits = ref<string[]>(Array(otpLength.value).fill(''))
const inputRefs = ref<HTMLInputElement[]>([])
const lastCompletedOtp = ref('')

const currentOtp = computed(() => digits.value.join(''))
const isComplete = computed(() => digits.value.every(Boolean) && currentOtp.value.length === otpLength.value)

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

const resetOtp = () => {
  digits.value = Array(otpLength.value).fill('')
  inputRefs.value = []
  lastCompletedOtp.value = ''
  emit('change', '')
}

const emitChange = () => {
  emit('change', currentOtp.value)

  if (!isComplete.value) {
    lastCompletedOtp.value = ''
    return
  }

  if (currentOtp.value !== lastCompletedOtp.value) {
    lastCompletedOtp.value = currentOtp.value
    emit('complete', currentOtp.value)
  }
}

const handleInput = (event: Event, index: number) => {
  const target = event.target as HTMLInputElement
  const value = target.value.replace(/\D/g, '')

  if (!value) {
    digits.value[index] = ''
    target.value = ''
    emitChange()
    return
  }

  digits.value[index] = value.slice(-1)
  target.value = digits.value[index]

  if (index < otpLength.value - 1) {
    focusInput(index + 1)
  }

  emitChange()
}

const handleKeydown = (event: KeyboardEvent, index: number) => {
  if (event.key !== 'Backspace') {
    return
  }

  event.preventDefault()

  if (digits.value[index]) {
    digits.value[index] = ''
    emitChange()
    return
  }

  if (index > 0) {
    digits.value[index - 1] = ''
    focusInput(index - 1)
    emitChange()
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
  emitChange()
}

watch(otpLength, resetOtp)
</script>

<template>
  <div class="w-full">
    <div
      class="flex justify-center gap-2 sm:gap-3"
      role="group"
      aria-label="OTP 驗證碼"
    >
      <input
        v-for="(_, index) in digits"
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

    <p
      v-if="props.error && props.errorMessage"
      id="otp-error-message"
      class="mt-3 text-center text-sm font-medium text-red-600"
    >
      {{ props.errorMessage }}
    </p>
  </div>
</template>
