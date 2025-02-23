<template>
  <v-container fluid fill-height class="pa-0 ma-0 align-center justify-center">
    <v-row no-gutters>
      <v-col cols="6" class="d-flex justify-center align-center mt-2" style="height: 100vh">
        <v-card class="pa-6" rounded="lg" elevation="3" style="width: 61.8%; height: 45%">
          <div class="text-h5 mb-2 text-center">Hello</div>
          <v-btn-toggle v-model="loginMethod" class="mb-6 justify-center">
            <v-btn value="password">密码登录</v-btn>
            <v-btn value="totp">TOTP 登录</v-btn>
          </v-btn-toggle>

          <v-text-field
              v-model="formData.name"
              label="用户名"
              variant="outlined"
              class="mb-4"
          />

          <!-- 密码登录方式 -->
          <div v-if="loginMethod === 'password'">
            <v-text-field
                v-model="formData.password"
                label="密码"
                type="password"
                variant="outlined"
                class="mb-4"
                @keyup.enter="handleLogin"
            />
          </div>

          <div v-else>
            <div class="d-flex justify-center">
              <v-otp-input
                  v-model="otpData">
              </v-otp-input>
            </div>
          </div>
          <div v-if="loginFeedback.message" class="text-center mt-2" :style="{ color: loginFeedback.color }">
            {{ loginFeedback.message }}
          </div>
          <!-- 登录按钮：点击同样触发验证 -->
          <v-btn color="primary" class="ma-0 mt-6" @click="handleLogin">
            登录
          </v-btn>
        </v-card>
      </v-col>
      <v-col cols="6" class="bg-grey" style="height: 100vh">

      </v-col>
    </v-row>
  </v-container>
</template>

<script setup lang="ts">
import {ref} from 'vue'

// 当前登录方式
const loginMethod = ref<'password' | 'totp'>('password')

// 表单数据
const formData = ref({
  name: '',
  password: '',
})

const otpData = ref('')

// 统一登录反馈（用于密码/TOTP均显示内联提示，格式：{ message, color }）
const loginFeedback = ref({message: '', color: ''})

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

/**
 * TOTP 验证函数：验证成功后显示绿色提示，再延时自动跳转
 */
function validateTOTP() {
  console.log(otpData.value)

  const user = users.find(u => u.name === formData.value.name)
  if (!user) {
    loginFeedback.value = {message: '用户不存在', color: 'red'}
    return
  }

  const isValid = true //TODO Replace it with real verify function

  if (isValid) {
    loginFeedback.value = {message: 'TOTP 验证成功', color: 'green'}
    setTimeout(() => {
      window.location.href = `https://www.bing.com/search?q=${user.nickname}`
    }, 2000)
  } else {
    loginFeedback.value = {message: 'TOTP 验证失败', color: 'red'}
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
    loginFeedback.value = {message: '用户不存在', color: 'red'}
    return
  }

  if (loginMethod.value === 'password') {
    if (formData.value.password === user.password) {
      loginFeedback.value = {message: '登陆成功', color: 'green'}
      setTimeout(() => {
        window.location.href = `https://www.bing.com/search?q=${user.nickname}`
      }, 2000)
    } else {
      loginFeedback.value = {message: '密码错误', color: 'red'}
    }
  } else if (loginMethod.value === 'totp') {
    // 若6位已填满，则调用自动验证逻辑
    validateTOTP()
  }
}

</script>

<style scoped>
</style>