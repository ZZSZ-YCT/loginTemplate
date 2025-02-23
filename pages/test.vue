<template>
  <v-container>
    <!-- 输入 Base32 密钥 -->
    <v-text-field
        v-model="secret"
        clearable
        label="请输入 Base32 密钥"
    ></v-text-field>
    <!-- 展示生成的 OTP（只读，自动刷新） -->
    <v-otp-input
        v-model="otp"
        focus-all
    ></v-otp-input>
  </v-container>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

// ---------- SJCL 及 TOTP 实现 Begin ----------

// 定义 sjcl 对象及部分异常
const sjcl: any = {
  cipher: {},
  hash: {},
  keyexchange: {},
  mode: {},
  misc: {},
  codec: {},
  exception: {
    corrupt: function(a: string) {
      this.toString = () => "CORRUPT: " + this.message;
      this.message = a;
    },
    invalid: function(a: string) {
      this.toString = () => "INVALID: " + this.message;
      this.message = a;
    },
    bug: function(a: string) {
      this.toString = () => "BUG: " + this.message;
      this.message = a;
    },
    notReady: function(a: string) {
      this.toString = () => "NOT READY: " + this.message;
      this.message = a;
    }
  }
};

sjcl.bitArray = {
  bitSlice: function(a: number[], b: number, c?: number): number[] {
    a = sjcl.bitArray.g(a.slice(Math.floor(b / 32)), 32 - (b & 31)).slice(1);
    return c === undefined ? a : sjcl.bitArray.clamp(a, c - b);
  },
  extract: function(a: number[], b: number, c: number): number {
    const d = Math.floor(-b - c & 31);
    return (
        (
            ((b + c - 1 ^ b) & -32)
                ? (a[Math.floor(b / 32)] << (32 - d)) ^ (a[Math.floor(b / 32) + 1] >>> d)
                : (a[Math.floor(b / 32)] >>> d)
        ) & ((1 << c) - 1)
    );
  },
  concat: function(a: number[], b: number[]): number[] {
    if (a.length === 0 || b.length === 0) return a.concat(b);
    const c = a[a.length - 1];
    const d = sjcl.bitArray.getPartial(c);
    return d === 32 ? a.concat(b) : sjcl.bitArray.g(b, d, c | 0, a.slice(0, a.length - 1));
  },
  bitLength: function(a: number[]): number {
    const b = a.length;
    return b === 0 ? 0 : 32 * (b - 1) + sjcl.bitArray.getPartial(a[b - 1]);
  },
  clamp: function(a: number[], b: number): number[] {
    if (32 * a.length < b) return a;
    a = a.slice(0, Math.ceil(b / 32));
    let c = a.length;
    b &= 31;
    if (c > 0 && b) {
      a[c - 1] = sjcl.bitArray.partial(b, a[c - 1] & (2147483648 >> (b - 1)), true);
    }
    return a;
  },
  partial: function(a: number, b: number, c?: boolean): number {
    return a === 32 ? b : (c ? b | 0 : b << (32 - a)) + 0x10000000000 * a;
  },
  getPartial: function(a: number): number {
    return Math.round(a / 0x10000000000) || 32;
  },
  equal: function(a: number[], b: number[]): boolean {
    if (sjcl.bitArray.bitLength(a) !== sjcl.bitArray.bitLength(b)) return false;
    let c = 0;
    for (let d = 0; d < a.length; d++) {
      c |= a[d] ^ b[d];
    }
    return c === 0;
  },
  g: function(a: number[], b: number, c: number, d?: number[]): number[] {
    let e = 0;
    if (d === undefined) {
      d = [];
    }
    for (; b >= 32; b -= 32) {
      d.push(c);
      c = 0;
    }
    if (b === 0) return d.concat(a);
    for (e = 0; e < a.length; e++) {
      d.push(c | (a[e] >>> b));
      c = a[e] << (32 - b);
    }
    let last = a.length ? a[a.length - 1] : 0;
    let partial = sjcl.bitArray.getPartial(last);
    d.push(sjcl.bitArray.partial((b + partial) & 31, (b + partial) > 32 ? c : d.pop()!, true));
    return d;
  },
  j: function(a: number[], b: number[]): number[] {
    return [a[0] ^ b[0], a[1] ^ b[1], a[2] ^ b[2], a[3] ^ b[3]];
  }
};

sjcl.codec.base32 = {
  e: "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567",
  fromBits: function(a: number[], pad?: boolean): string {
    let c = "";
    let d = 0, e = 0;
    const g = sjcl.codec.base32.e;
    let f = 0;
    const k = sjcl.bitArray.bitLength(a);
    for (d = 0; 5 * c.length < k;) {
      c += g.charAt(((e ^ (a[d] >>> e)) >>> 27));
      if (5 > e) {
        e += 27;
        f = a[d] << (5 - e);
        d++;
      } else {
        e -= 5;
      }
    }
    while ((c.length & 5) && !pad) {
      c += "=";
    }
    return c;
  },
  toBits: function(a: string): number[] {
    a = a.replace(/\s|=/g, "").toUpperCase();
    const b: number[] = [];
    let c = 0, d = 0;
    let g = 0;
    let f: number;
    for (c = 0; c < a.length; c++) {
      f = sjcl.codec.base32.e.indexOf(a.charAt(c));
      if (f < 0) throw new sjcl.exception.invalid("this isn't base32!");
      if (d > 27) {
        d -= 27;
        b.push(g ^ (f >>> d));
        g = f << (32 - d);
      } else {
        d += 5;
        g ^= f << (32 - d);
      }
    }
    if (d & 56) {
      b.push(sjcl.bitArray.partial(d & 56, g, true));
    }
    return b;
  }
};

sjcl.codec.utf8String = {
  toBits: function(str: string): number[] {
    const encoder = new TextEncoder();
    const bytes = encoder.encode(str);
    const words: number[] = [];
    for (let i = 0; i < bytes.length; i += 4) {
      words.push(
          (bytes[i] << 24) |
          ((bytes[i + 1] || 0) << 16) |
          ((bytes[i + 2] || 0) << 8) |
          (bytes[i + 3] || 0)
      );
    }
    return words;
  }
};

sjcl.hash.sha1 = function(this: any, a?: any) {
  if (a) {
    this.d = a.d.slice(0);
    this.b = a.b.slice(0);
    this.a = a.a;
  } else {
    this.reset();
  }
};
sjcl.hash.sha1.hash = function(a: any): number[] {
  return new (sjcl.hash.sha1 as any)().update(a).finalize();
};

sjcl.hash.sha1.prototype = {
  blockSize: 512,
  reset: function(this: any) {
    this.d = this.h.slice(0);
    this.b = [];
    this.a = 0;
    return this;
  },
  update: function(this: any, a: string | number[]): any {
    if (typeof a === "string") {
      a = sjcl.codec.utf8String.toBits(a);
    }
    this.b = sjcl.bitArray.concat(this.b, a);
    const b = this.a;
    this.a = b + sjcl.bitArray.bitLength(a);
    for (let c = (this.blockSize + b) & -this.blockSize; c <= this.a; c += this.blockSize) {
      n(this, this.b.splice(0, 16));
    }
    return this;
  },
  finalize: function(this: any): number[] {
    let a;
    let b = this.b;
    const c = this.d;
    b = sjcl.bitArray.concat(b, [sjcl.bitArray.partial(1, 1)]);
    for (a = b.length + 2; a & 15; a++) {
      b.push(0);
    }
    b.push(Math.floor(this.a / 0x100000000));
    b.push(this.a | 0);
    while (b.length) {
      n(this, b.splice(0, 16));
    }
    this.reset();
    return c;
  },
  h: [1732584193, 4023233417, 2562383102, 271733878, 3285377520],
  i: [1518500249, 1859775393, 2400959708, 3395469782]
};

function n(a: any, b: number[]): void {
  let c: number, d: number, e: number;
  let g: number, f: number, k: number, m: number;
  let l = b.slice(0);
  let h = a.d;
  let e0 = h[0], g0 = h[1], f0 = h[2], k0 = h[3], m0 = h[4];
  for (c = 0; c <= 79; c++) {
    if (c >= 16) {
      l[c] =
          ((l[c - 3] ^ l[c - 8] ^ l[c - 14] ^ l[c - 16]) << 1) |
          ((l[c - 3] ^ l[c - 8] ^ l[c - 14] ^ l[c - 16]) >>> 31);
    }
    if (c <= 19) {
      d = (g0 & f0) | ((~g0) & k0);
    } else if (c <= 39) {
      d = g0 ^ f0 ^ k0;
    } else if (c <= 59) {
      d = (g0 & f0) | (g0 & k0) | (f0 & k0);
    } else {
      d = g0 ^ f0 ^ k0;
    }
    d = (((e0 << 5) | (e0 >>> 27)) + d + m0 + l[c] + a.i[Math.floor(c / 20)]) | 0;
    m0 = k0;
    k0 = f0;
    f0 = (g0 << 30) | (g0 >>> 2);
    g0 = e0;
    e0 = d;
  }
  h[0] = (h[0] + e0) | 0;
  h[1] = (h[1] + g0) | 0;
  h[2] = (h[2] + f0) | 0;
  h[3] = (h[3] + k0) | 0;
  h[4] = (h[4] + m0) | 0;
}

sjcl.misc.hmac = function(this: any, a: number[], b?: any) {
  this.f = b = b || sjcl.hash.sha256;
  const c: number[][] = [[], []];
  const e = b.prototype.blockSize / 32;
  this.c = [new b(), new b()];
  if (a.length > e) a = b.hash(a);
  for (let d = 0; d < e; d++) {
    c[0][d] = a[d] ^ 909522486;
    c[1][d] = a[d] ^ 1549556828;
  }
  this.c[0].update(c[0]);
  this.c[1].update(c[1]);
};

sjcl.misc.hmac.prototype.encrypt = sjcl.misc.hmac.prototype.mac = function(this: any, a: any): number[] {
  a = new this.f(this.c[0]).update(a).finalize();
  return new this.f(this.c[1]).update(a).finalize();
};

// TOTP 函数：输入 Base32 密钥，输出 6 位动态验证码
function TOTP(K: string): string {
  const C = Math.floor(new Date().getTime() / 30000);
  const key = sjcl.codec.base32.toBits(K);
  // count 为 64 位，但 JS 位运算只处理 32 位整数
  const count = [((C & 0xffffffff00000000) >> 32), C & 0xffffffff];
  const otplength = 6;
  const hmacsha1 = new sjcl.misc.hmac(key, sjcl.hash.sha1);
  const code = hmacsha1.encrypt(count);
  const offset = sjcl.bitArray.extract(code, 152, 8) & 0x0f;
  const startBits = offset * 8;
  const slice = sjcl.bitArray.bitSlice(code, startBits, startBits + 32);
  const dbc1 = slice[0];
  const dbc2 = dbc1 & 0x7fffffff;
  const otp = dbc2 % Math.pow(10, otplength);
  let result = otp.toString();
  while (result.length < otplength) {
    result = "0" + result;
  }
  return result;
}

// ---------- SJCL 及 TOTP 实现 End ----------

// 响应式变量：secret 存储用户输入的 Base32 密钥，otp 保存生成的验证码
const secret = ref("JBSWY3DPEHPK3PXP");
const otp = ref("");

// 更新 OTP 的函数
function updateOTP() {
  if (secret.value.trim() !== "") {
    try {
      otp.value = TOTP(secret.value.trim());
    } catch (error) {
      otp.value = "错误";
    }
  } else {
    otp.value = "";
  }
}

// 初始更新一次
updateOTP();

let timer: ReturnType<typeof setInterval>;

// 仅在客户端挂载后启动定时器，避免服务端执行 setInterval
onMounted(() => {
  timer = setInterval(updateOTP, 1000);
});

onUnmounted(() => {
  clearInterval(timer);
});
</script>
