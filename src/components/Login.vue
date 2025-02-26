<template class="bg-red">
    <div v-if="!user">
      <h2>Login</h2>
      <input v-model="email" placeholder="E-mail" />
      <input v-model="password" type="password" placeholder="Senha" />
      <button @click="login">Entrar</button>
      <p v-if="authError" class="error">{{ authError }}</p>
    </div>
  </template>
  
  <script setup>
  import { ref } from 'vue';
  import supabase from '../supabase';
  
  const email = ref('');
  const password = ref('');
  const authError = ref('');
  const user = ref(null);
  
  const login = async () => {
    authError.value = '';
    const { data, error } = await supabase.auth.signInWithPassword({
      email: email.value,
      password: password.value
    });
  
    if (error) {
      authError.value = 'Falha ao logar. Verifique os dados.';
    } else {
      user.value = data.user;
    }
  };
  
  const logout = async () => {
    await supabase.auth.signOut();
    user.value = null;
  };
  </script>
  
  <style scoped>
  .error {
    color: red;
  }
  </style>
  