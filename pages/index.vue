<script setup lang="ts">
interface VerifyOtpResponse {
  success: boolean
  data?: {
    verified: boolean
    timestamp: string
  }
}

const isLoading = ref(false)
const isVerified = ref(false)
const hasError = ref(false)
const errorMessage = ref('')
const otp = ref('')
const otpLength = 6
const otpItemsPerRow = 2
const lastVerifiedOtp = ref('')

const resetStatus = () => {
  isVerified.value = false
  hasError.value = false
  errorMessage.value = ''
}

const verifyOtp = async (value: string) => {
  if (isLoading.value) {
    return
  }

  isLoading.value = true
  lastVerifiedOtp.value = value

  try {
    const response = await $fetch<VerifyOtpResponse>('/api/examples/verify-otp-simple', {
      method: 'POST',
      body: { otp: value }
    })

    const verified = response.success || Boolean(response.data?.verified)

    if (verified) {
      isVerified.value = true
      return
    }

    hasError.value = true
    errorMessage.value = '驗證碼錯誤，請重新輸入'
  } catch {
    hasError.value = true
    errorMessage.value = '驗證失敗，請稍後再試'
  } finally {
    isLoading.value = false
  }
}

watch(otp, (value) => {
  resetStatus()

  if (value.length !== otpLength) {
    lastVerifiedOtp.value = ''
    return
  }

  if (value === lastVerifiedOtp.value) {
    return
  }

  verifyOtp(value)
})
</script>

<template>
  <main class="flex min-h-screen items-center justify-center bg-slate-50 px-4 py-10 text-slate-950">
    <section class="w-full max-w-md">
      <div class="mb-8 text-center">
        <h1 class="text-2xl font-bold">
          OTP 驗證
        </h1>
        <p class="mt-2 text-sm text-slate-600">
          請輸入 6 位數驗證碼
        </p>
      </div>

      <BaseInputOtp
        v-model="otp"
        :length="otpLength"
        :n="otpItemsPerRow"
        :error="hasError"
        :error-message="errorMessage"
        :disabled="isLoading"
      />

      <p
        v-if="isLoading"
        class="mt-4 text-center text-sm font-medium text-slate-600"
      >
        驗證中...
      </p>

      <p
        v-else-if="isVerified"
        class="mt-4 text-center text-sm font-semibold text-emerald-600"
      >
        驗證成功
      </p>
    </section>
  </main>
</template>
