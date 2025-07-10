<template>
  <div class="step">
    <div class="step-title">
      <span class="step-number">1</span>
      連接 Spotify 帳戶
    </div>
    <p class="step-description">授權存取您的 Spotify 帳戶</p>
    
    <div class="auth-buttons">
      <button 
        class="btn btn-spotify" 
        style="display:inline-flex; align-items:center;"
        @click="authenticate"
        :disabled="isAuthenticating || isAuthenticated"
      >
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" style="margin-right: 8px" fill="currentColor" class="bi bi-spotify" viewBox="0 0 16 16"><path d="M8 0a8 8 0 1 0 0 16A8 8 0 0 0 8 0m3.669 11.538a.5.5 0 0 1-.686.165c-1.879-1.147-4.243-1.407-7.028-.77a.499.499 0 0 1-.222-.973c3.048-.696 5.662-.397 7.77.892a.5.5 0 0 1 .166.686m.979-2.178a.624.624 0 0 1-.858.205c-2.15-1.321-5.428-1.704-7.972-.932a.625.625 0 0 1-.362-1.194c2.905-.881 6.517-.454 8.986 1.063a.624.624 0 0 1 .206.858m.084-2.268C10.154 5.56 5.9 5.419 3.438 6.166a.748.748 0 1 1-.434-1.432c2.825-.857 7.523-.692 10.492 1.07a.747.747 0 1 1-.764 1.288"/></svg>
        {{ buttonText }}
      </button>
      
      <button 
        v-if="isAuthenticated"
        class="btn btn-logout"
        style="display:inline-flex; align-items:center;"
        @click="logout"
      >
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" style="margin-right: 8px;" class="bi bi-box-arrow-right" viewBox="0 0 16 16"><path fill-rule="evenodd" d="M10 15a1 1 0 0 0 1-1v-3h-1v3H2V2h8v3h1V2a1 1 0 0 0-1-1H2a1 1 0 0 0-1 1v12a1 1 0 0 0 1 1h8z"/><path fill-rule="evenodd" d="M15.854 8.354a.5.5 0 0 0 0-.708l-3-3a.5.5 0 0 0-.708.708L14.293 7.5H5.5a.5.5 0 0 0 0 1h8.793l-2.147 2.146a.5.5 0 0 0 .708.708l3-3z"/></svg>
        登出
      </button>
    </div>
    
    <div 
      v-if="showStatus" 
      :class="['auth-status', isAuthenticated ? 'auth-success' : 'auth-pending']"
    >
      {{ statusText }}
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'

const emit = defineEmits(['authenticated', 'logout'])

const isAuthenticated = ref(false)
const isAuthenticating = ref(false)
const showStatus = ref(false)
const userName = ref('');

const clientId = import.meta.env.VITE_SPOTIFY_CLIENT_ID;
const redirectUrl = import.meta.env.VITE_REDIRECT_URL || 'http://127.0.0.1:8888';
const authorizationEndpoint = "https://accounts.spotify.com/authorize";
const tokenEndpoint = "https://accounts.spotify.com/api/token";
const scope = 'playlist-modify-public playlist-modify-private';

const buttonText = computed(() => {
  if (isAuthenticated.value) return '已連接 Spotify'
  if (isAuthenticating.value) return '連接中...'
  return '連接 Spotify 帳戶'
})

const statusText = computed(() => {
  return isAuthenticated.value 
    ? `已成功連接 Spotify 帳戶：${userName.value || '未知用戶'}`
    : '正在等待 Spotify 授權...'
})

watch(isAuthenticated, async (newVal) => {
  if (newVal) {
    const data = await getUserData();
    userName.value = data?.display_name || '未知用戶';
  }
})


const currentToken = {
  get access_token() { return localStorage.getItem('access_token') || null; },
  get refresh_token() { return localStorage.getItem('refresh_token') || null; },
  get expires_in() { return localStorage.getItem('refresh_in') || null },
  get expires() { return localStorage.getItem('expires') || null },

  save: function (response) {
    const { access_token, refresh_token, expires_in } = response;
    localStorage.setItem('access_token', access_token);
    localStorage.setItem('refresh_token', refresh_token);
    localStorage.setItem('expires_in', expires_in);

    const now = new Date();
    const expiry = new Date(now.getTime() + (expires_in * 1000));
    localStorage.setItem('expires', expiry);
  }
};

async function checkToken() {
  let isTokenValid = false;
  isAuthenticating.value = true;

  // On page load, try to fetch auth code from current browser search URL
  const args = new URLSearchParams(window.location.search);
  const code = args.get('code');

  // If we find a code, we're in a callback, do a token exchange
  if (code) {
    const token = await getToken(code);
    currentToken.save(token);

    // Remove code from URL so we can refresh correctly.
    const url = new URL(window.location.href);
    url.searchParams.delete("code");

    const updatedUrl = url.search ? url.href : url.href.replace('?', '');
    window.history.replaceState({}, document.title, updatedUrl);
  }

  // If we have a token, we're logged in, so fetch user data and render logged in template
  if (currentToken.access_token) {
    const expires = new Date(currentToken.expires);
    const now = new Date();
    if (expires <= now) {
      console.log("Access token expired, trying to refresh...");
      const refreshed = await refreshToken();
      if (refreshed) {
        isTokenValid = true;
      } else {
        console.warn("Refresh failed, login again");
        isTokenValid = false;
      }
    } else {
      isTokenValid = true;
    }
  // Otherwise we're not logged in, so render the login template
  } else {
    isTokenValid = false;
  }

  isAuthenticating.value = false;
  if(isTokenValid) {
    isAuthenticated.value = true;
    showStatus.value = true;
    emit('authenticated', currentToken.access_token);
  } else {
    isAuthenticated.value = false;
    showStatus.value = false;
  }
  
}

async function authenticate() {
  if (typeof window === 'undefined' || !window.crypto) {
    console.error('Crypto not available during build.');
    return;
  }

  const possible = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  const randomValues = window.crypto.getRandomValues(new Uint8Array(64));
  const randomString = randomValues.reduce((acc, x) => acc + possible[x % possible.length], "");
  const code_verifier = randomString;
  const data = new TextEncoder().encode(code_verifier);
  const hashed = await window.crypto.subtle.digest('SHA-256', data);

  const code_challenge_base64 = btoa(String.fromCharCode(...new Uint8Array(hashed)))
    .replace(/=/g, '')
    .replace(/\+/g, '-')
    .replace(/\//g, '_');

  window.localStorage.setItem('code_verifier', code_verifier);

  const authUrl = new URL(authorizationEndpoint)
  const params = {
    response_type: 'code',
    client_id: clientId,
    scope: scope,
    code_challenge_method: 'S256',
    code_challenge: code_challenge_base64,
    redirect_uri: redirectUrl,
  };

  authUrl.search = new URLSearchParams(params).toString();
  window.location.href = authUrl.toString(); // Redirect the user to the authorization server for login
}

async function getToken(code) {
  const code_verifier = localStorage.getItem('code_verifier');

  const response = await fetch(tokenEndpoint, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: new URLSearchParams({
      client_id: clientId,
      grant_type: 'authorization_code',
      code: code,
      redirect_uri: redirectUrl,
      code_verifier: code_verifier,
    }),
  });

  return await response.json();
}

async function refreshToken() {
  const response = await fetch(tokenEndpoint, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams({
      client_id: clientId,
      grant_type: 'refresh_token',
      refresh_token: currentToken.refresh_token
    }),
  });

  if (!response.ok) {
    console.error("Failed to refresh token:", response.status, response.statusText);
    return false;
  }

  const tokenData = await response.json();
  currentToken.save({
    access_token: tokenData.access_token,
    refresh_token: tokenData.refresh_token || currentToken.refresh_token,
    expires_in: tokenData.expires_in,
  });
  console.log("Successfully refreshed access token.");
  return true;
}

async function getUserData() {
  const response = await fetch('https://api.spotify.com/v1/me', {
    headers: {
      'Authorization': `Bearer ${currentToken.access_token}`
    }
  });

  if (!response.ok) {
    console.error("Failed to fetch user data:", response.status, response.statusText);
    return null;
  }

  return await response.json();
}

function logout() {
  localStorage.removeItem('access_token');
  localStorage.removeItem('refresh_token');
  localStorage.removeItem('expires_in');
  localStorage.removeItem('expires');
  localStorage.removeItem('code_verifier');
  
  isAuthenticated.value = false;
  isAuthenticating.value = false;
  showStatus.value = false;
  
  emit('logout');
}

onMounted(async () => {
  await checkToken();
})

</script>

<style scoped>
.step {
  margin-bottom: 30px;
  padding: 20px;
  border-radius: 12px;
  border: 2px solid #f0f0f0;
  transition: all 0.3s ease;
}

.step:hover {
  border-color: #1db954;
  transform: translateY(-2px);
}

.step-number {
  display: inline-block;
  width: 30px;
  height: 30px;
  background: #2d3974;
  color: white;
  border-radius: 50%;
  text-align: center;
  line-height: 30px;
  font-weight: bold;
  margin-right: 15px;
}

.step-title {
  font-size: 1.2rem;
  font-weight: bold;
  margin-bottom: 10px;
  color: #333;
}

.step-description {
  margin-bottom: 15px;
  color: #666;
}

.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  margin-right: 10px;
}

.auth-buttons {
  display: flex;
  gap: 10px;
  align-items: center;
  flex-wrap: wrap;
}

.btn-spotify {
  background: #1db954;
  color: white;
}

.btn-spotify:hover:not(:disabled) {
  background: #1ed760;
  transform: translateY(-2px);
}

.btn-logout {
  background: #dc3545;
  color: white;
}

.btn-logout:hover {
  background: #c82333;
  transform: translateY(-2px);
}

.btn:disabled {
  background: #ccc;
  cursor: not-allowed;
  transform: none;
}

.auth-status {
  padding: 15px;
  border-radius: 8px;
  margin: 15px 0;
  font-weight: 500;
}

.auth-success {
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.auth-pending {
  background: #fff3cd;
  color: #856404;
  border: 1px solid #ffeaa7;
}
</style>