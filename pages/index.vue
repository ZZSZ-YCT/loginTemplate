<template>
  <v-container fluid class="d-flex flex-column align-center justify-center login-container">
    <!-- 登录卡片 -->
    <v-card class="login-card pa-6" rounded="lg" elevation="3">
      <div class="text-h5 mb-6 text-center">Hello</div>

      <!-- 分段切换：密码登录 / TOTP 登录 -->
      <v-btn-toggle v-model="loginMethod" mandatory class="mb-6 justify-center">
        <v-btn value="password">密码登录</v-btn>
        <v-btn value="totp">TOTP 登录</v-btn>
      </v-btn-toggle>

      <!-- 用户名 -->
      <v-text-field
          v-model="formData.name"
          label="用户名"
          variant="outlined"
          hide-details
          class="mb-4"
      />

      <!-- 密码登录方式 -->
      <div v-if="loginMethod === 'password'">
        <v-text-field
            v-model="formData.password"
            label="密码"
            type="password"
            variant="outlined"
            hide-details
            class="mb-4"
            @keyup.enter="handleLogin"
        />
        <!-- 密码登录反馈提示 -->
        <div v-if="loginFeedback.message" class="text-center mt-2" :style="{ color: loginFeedback.color }">
          {{ loginFeedback.message }}
        </div>
      </div>

      <!-- TOTP 登录方式 -->
      <div v-else>
        <!-- 6个方形输入框：支持粘贴、实时验证 -->
        <div class="d-flex justify-center totp-wrapper">
          <v-text-field
              v-for="(digit, index) in totpDigits"
              :key="index"
              v-model="totpDigits[index]"
              variant="outlined"
              hide-details
              class="totp-input"
              maxlength="1"
              @input="onInput(index)"
              @keydown.backspace="onBackspace(index, $event)"
              @paste="onPaste($event, index)"
              type="tel"
              inputmode="numeric"
              :ref="el => totpInputRefs[index] = el"
          />
        </div>
        <!-- TOTP 自动验证后的反馈提示（仅当6位填满时显示） -->
        <div v-if="totpFilled" class="text-center mt-2" :style="{ color: loginFeedback.color }">
          {{ loginFeedback.message }}
        </div>
      </div>

      <!-- 登录按钮：点击同样触发验证 -->
      <v-btn color="primary" class="ma-0 mt-6" @click="handleLogin">
        登录
      </v-btn>
    </v-card>
  </v-container>
</template>

<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import { authenticator } from 'otplib'

// 当前登录方式
const loginMethod = ref<'password' | 'totp'>('password')

// 表单数据
const formData = ref({
  name: '',
  password: '',
})

// TOTP 输入的 6 位
const totpDigits = ref(['', '', '', '', '', ''])

// TOTP 输入框的引用数组（用于聚焦操作）
const totpInputRefs: Array<any> = []

// 统一登录反馈（用于密码/TOTP均显示内联提示，格式：{ message, color }）
const loginFeedback = ref({ message: '', color: '' })

// 示例用户数据（前端演示用，真实项目中应由后端验证）
interface User {
  name: string
  nickname: string
  password: string
  totpSecret: string
}

const users: User[] = [
  {
    name: 'alice',
    nickname: 'AliceInWonderland',
    password: '123456',
    totpSecret: 'JBSWY3DPEHPK3PXP',
  },
  {
    name: 'bob',
    nickname: 'BobbyBrown',
    password: 'abcdef',
    totpSecret: 'KB2WY3DPEHPK3PXQ',
  },
  {
    name: 'charlie',
    nickname: 'CharliePuth',
    password: 'p@ssw0rd',
    totpSecret: 'NB3WY3DPEHPK3PXO',
  },
]

// 计算 TOTP 是否已完整填满（6个输入框均填1个字符）
const totpFilled = computed(() => totpDigits.value.every(digit => digit.length === 1))

// 监听 TOTP 输入变化：当6位填满时自动验证；否则清除反馈
watch(totpDigits, () => {
  if (loginMethod.value === 'totp' && totpFilled.value) {
    validateTOTP()
  } else if (loginMethod.value === 'totp') {
    loginFeedback.value = { message: '', color: '' }
  }
}, { deep: true })

// 切换登录方式时，清空对应输入与反馈
watch(loginMethod, (newMethod) => {
  loginFeedback.value = { message: '', color: '' }
  if (newMethod === 'totp') {
    totpDigits.value = ['', '', '', '', '', '']
  } else if (newMethod === 'password') {
    formData.value.password = ''
  }
})

/**
 * TOTP 验证函数：验证成功后显示绿色提示，再延时自动跳转
 */
function validateTOTP() {
  const user = users.find(u => u.name === formData.value.name)
  if (!user) {
    loginFeedback.value = { message: '用户不存在', color: 'red' }
    return
  }
  const totpToken = totpDigits.value.join('')
  const isValid = authenticator.verify({ token: totpToken, secret: user.totpSecret })
  if (isValid) {
    loginFeedback.value = { message: 'TOTP 验证成功', color: 'green' }
    setTimeout(() => {
      window.location.href = `https://www.bing.com/search?q=${user.nickname}`
    }, 2000)
  } else {
    loginFeedback.value = { message: 'TOTP 验证失败', color: 'red' }
  }
}

/**
 * 登录统一处理：密码或 TOTP 都在此验证
 * - 密码：按下 Enter 或点击按钮时触发，验证后显示反馈提示
 * - TOTP：若用户点击登录按钮，也会触发一次验证（通常6位自动验证已处理）
 */
function handleLogin() {
  const user = users.find(u => u.name === formData.value.name)
  if (!user) {
    loginFeedback.value = { message: '用户不存在', color: 'red' }
    return
  }

  if (loginMethod.value === 'password') {
    if (formData.value.password === user.password) {
      loginFeedback.value = { message: '登陆成功', color: 'green' }
      setTimeout(() => {
        window.location.href = `https://www.bing.com/search?q=${user.nickname}`
      }, 2000)
    } else {
      loginFeedback.value = { message: '密码错误', color: 'red' }
    }
  } else if (loginMethod.value === 'totp') {
    if (!totpFilled.value) {
      loginFeedback.value = { message: '请先完整输入6位TOTP', color: 'red' }
      return
    }
    // 若6位已填满，则调用自动验证逻辑
    validateTOTP()
  }
}

/**
 * 每个 TOTP 输入框输入时：
 * - 限制只保留最后一个字符
 * - 输入有效后自动聚焦下一输入框
 */
function onInput(index: number) {
  let currentVal = totpDigits.value[index]
  if (currentVal.length > 1) {
    totpDigits.value[index] = currentVal.slice(-1)
  }
  if (totpDigits.value[index] !== '' && index < totpDigits.value.length - 1) {
    focusField(index + 1)
  }
}

/**
 * 监听 backspace：若当前输入框为空，则删除上一个并聚焦
 */
function onBackspace(index: number, event: KeyboardEvent) {
  if (totpDigits.value[index] === '' && index > 0) {
    event.preventDefault()
    totpDigits.value[index - 1] = ''
    focusField(index - 1)
  }
}

/**
 * 处理粘贴事件：提取粘贴文本中的数字，依次填入每个输入框
 */
function onPaste(event: ClipboardEvent, index: number) {
  event.preventDefault()
  const clipboardData = event.clipboardData?.getData('text') || ''
  // 只保留数字，最多取6位
  const digits = clipboardData.replace(/\D/g, '').slice(0, totpDigits.value.length)
  if (!digits) return

  let idx = index
  for (let i = 0; i < digits.length && idx < totpDigits.value.length; i++, idx++) {
    totpDigits.value[idx] = digits[i]
  }
  if (idx < totpDigits.value.length) {
    focusField(idx)
  }
}

/**
 * 聚焦到指定的 TOTP 输入框
 */
function focusField(index: number) {
  totpInputRefs[index]?.focus?.()
}
</script>

<style scoped>
.login-container {
  min-height: 100vh;
  background: #f0f2f5;
}
.login-card {
  width: 340px;
  border-radius: 24px !important;
  background-color: #fff;
}
.totp-wrapper {
  display: flex;
  gap: 8px;
}
.totp-input {
  width: 48px;
  text-align: center;
}
.totp-input input {
  text-align: center;
  height: 48px !important;
  margin: 0 auto;
  padding: 0;
}
</style>