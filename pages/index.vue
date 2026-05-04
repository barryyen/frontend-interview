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

const resetStatus = () => {
  isVerified.value = false
  hasError.value = false
  errorMessage.value = ''
}

const handleOtpChange = () => {
  resetStatus()
}

const handleOtpComplete = async (otp: string) => {
  if (isLoading.value) {
    return
  }

  resetStatus()
  isLoading.value = true

  try {
    const response = await $fetch<VerifyOtpResponse>('/api/examples/verify-otp-simple', {
      method: 'POST',
      body: { otp }
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
        :length="6"
        :error="hasError"
        :error-message="errorMessage"
        :disabled="isLoading"
        @change="handleOtpChange"
        @complete="handleOtpComplete"
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
