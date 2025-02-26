<template>
    <div>
      <h3>Links de Eventos Criados</h3>
      <ul>
        <li v-for="(link, index) in eventosCriados" :key="index">
          <a :href="link" target="_blank">Adicionar Evento no Google Calendar</a>
          <button @click="copiarLink(link)">Copiar Link</button>
        </li>
      </ul>
    </div>
  </template>
  
  <script setup>
  import { defineProps } from 'vue';
  
  const eventosCriados = defineProps({
    eventosCriados: Array,
  });
  
  const copiarLink = (link) => {
    navigator.clipboard.writeText(link).then(() => {
      alert('Link copiado!');
    });
  };
  </script>
  