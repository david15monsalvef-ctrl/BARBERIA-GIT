<script setup>
import { ref, computed } from 'vue'
import { useLocalStorage } from '@vueuse/core'

// Persistencia de los datos con @vueuse/core
const servicios = useLocalStorage('barberia_servicios_v2', [
  {
    id: 1,
    cliente: 'Carlos Gómez',
    serviciosSeleccionados: ['Corte + Barba'],
    barbero: 'Don Ramiro',
    fecha: '2026-09-07T10:30',
    precio: 35000,
    metodoPago: 'efectivo',
    estadoPago: 'pagado',
    observaciones: 'Cliente habitual, degradado bajo.'
  },
  {
    id: 2,
    cliente: 'Andrés Morales',
    serviciosSeleccionados: ['Corte moderno + Barba'],
    barbero: 'Mateo',
    fecha: '2026-09-07T11:15',
    precio: 40000,
    metodoPago: 'transferencia',
    estadoPago: 'pendiente',
    observaciones: 'Pendiente comprobante Nequi.'
  }
])

// Catálogo de servicios disponibles con sus precios base
const catalogoServicios = [
  { nombre: 'Corte clásico', precio: 20000 },
  { nombre: 'Corte moderno', precio: 25000 },
  { nombre: 'Barba', precio: 15000 },
  { nombre: 'Corte + Barba', precio: 35000 },
  { nombre: 'Corte moderno + Barba', precio: 40000 },
  { nombre: 'Cejas', precio: 8000 },
  { nombre: 'Tinte', precio: 40000 }
]

// Estados para modales y control
const mostrarModal = ref(false)
const modoEdicion = ref(false)
const idEdicion = ref(null)
const servicioAEliminar = ref(null)

// Formulario reactivo
const formulario = ref({
  cliente: '',
  serviciosSeleccionados: ['Corte clásico'],
  barbero: 'Don Ramiro',
  fecha: '',
  metodoPago: 'efectivo',
  estadoPago: 'pendiente',
  observaciones: ''
})

// Propiedades computadas para las métricas del panel superior
const totalServicios = computed(() => servicios.value.length)

const ventasTotales = computed(() => {
  return servicios.value.reduce((acc, s) => acc + (s.precio || 0), 0)
})

const dineroPendiente = computed(() => {
  return servicios.value
    .filter(s => s.estadoPago === 'pendiente' || s.estadoPago === 'fiado')
    .reduce((acc, s) => acc + (s.precio || 0), 0)
})

// Cálculo automático del precio en base a los servicios elegidos
const precioCalculado = computed(() => {
  return formulario.value.serviciosSeleccionados.reduce((total, nombreServicio) => {
    const encontrado = catalogoServicios.find(s => s.nombre === nombreServicio)
    return total + (encontrado ? encontrado.precio : 0)
  }, 0)
})

function abrirModalCrear() {
  modoEdicion.value = false
  idEdicion.value = null
  const ahora = new Date()
  ahora.setMinutes(ahora.getMinutes() - ahora.getTimezoneOffset())
  
  formulario.value = {
    cliente: '',
    serviciosSeleccionados: ['Corte clásico'],
    barbero: 'Don Ramiro',
    fecha: ahora.toISOString().slice(0, 16),
    metodoPago: 'efectivo',
    estadoPago: 'pendiente',
    observaciones: ''
  }
  mostrarModal.value = true
}

function abrirModalEditar(item) {
  modoEdicion.value = true
  idEdicion.value = item.id
  const listaServicios = item.serviciosSeleccionados || [item.servicio || 'Corte clásico']
  formulario.value = { 
    ...item, 
    serviciosSeleccionados: [...listaServicios] 
  }
  mostrarModal.value = true
}

function cerrarModal() {
  mostrarModal.value = false
}

function guardarServicio() {
  if (!formulario.value.cliente.trim()) {
    alert('Por favor ingrese el nombre del cliente.')
    return
  }

  if (formulario.value.serviciosSeleccionados.length === 0) {
    alert('Debe seleccionar al menos un servicio.')
    return
  }

  const datosServicio = {
    ...formulario.value,
    precio: precioCalculado.value
  }

  if (modoEdicion.value) {
    for (let i = 0; i < servicios.value.length; i++) {
      if (servicios.value[i].id === idEdicion.value) {
        servicios.value[i] = { ...datosServicio, id: idEdicion.value }
        break
      }
    }
  } else {
    servicios.value.unshift({
      ...datosServicio,
      id: Date.now()
    })
  }
  cerrarModal()
}

function pedirConfirmacionEliminar(servicio) {
  servicioAEliminar.value = servicio
}

function borrarServicio() {
  if (servicioAEliminar.value) {
    servicios.value = servicios.value.filter(s => s.id !== servicioAEliminar.value.id)
    servicioAEliminar.value = null
  }
}
</script>

<template>
  <div class="contenedor-dashboard">
    <!-- Header Principal -->
    <header class="header">
      <div class="header-info">
        <h1>💈 Barbería Don Ramiro</h1>
        <p>Registro de servicios</p>
      </div>
      <button class="btn btn-primario" @click="abrirModalCrear">+ Registrar servicio</button>
    </header>

    <!-- Barra de Métricas Financieras y Estadísticas -->
    <section class="metrics-bar">
      <div class="metric-card">
        <span class="metric-title">Servicios</span>
        <span class="metric-value">{{ totalServicios }}</span>
      </div>
      <div class="metric-card">
        <span class="metric-title">Ventas totales</span>
        <span class="metric-value">${{ ventasTotales.toLocaleString() }}</span>
      </div>
      <div class="metric-card">
        <span class="metric-title">Dinero pendiente</span>
        <span class="metric-value text-warning">${{ dineroPendiente.toLocaleString() }}</span>
      </div>
    </section>

    <h2 class="section-title">Servicios registrados</h2>

    <!-- Estado Vacío -->
    <div v-if="servicios.length === 0" class="vacio">
      <p>No hay servicios registrados en este momento.</p>
    </div>

    <!-- Grid de Tarjetas (Aprovechando el ancho de pantalla) -->
    <main v-else class="grid-amplio">
      <div 
        v-for="s in servicios" 
        :key="s.id" 
        class="card"
        :class="{ 'card-fiado': s.estadoPago === 'fiado' }"
      >
        <div class="card-head">
          <h3>{{ s.cliente }}</h3>
          <span class="badge" :class="s.estadoPago">
            <span v-if="s.estadoPago === 'pagado'">✅ Pagado</span>
            <span v-else-if="s.estadoPago === 'pendiente'">⏳ Pendiente</span>
            <span v-else>⚠️ Fiado</span>
          </span>
        </div>

        <div class="card-body">
          <p><strong>Servicios:</strong> 
            <span class="tag-servicio" v-for="(serv, idx) in (s.serviciosSeleccionados || [s.servicio])" :key="idx">
              {{ serv }}
            </span>
          </p>
          <p><strong>Barbero:</strong> ✂️ {{ s.barbero }}</p>
          <p><strong>Fecha y Hora:</strong> 📅 {{ new Date(s.fecha).toLocaleString() }}</p>
          <p class="precio-destacado"><strong>Total:</strong> ${{ s.precio.toLocaleString() }}</p>
          <p>
            <strong>Método de Pago:</strong> 
            <span v-if="s.metodoPago === 'efectivo'">💵 Efectivo</span>
            <span v-else-if="s.metodoPago === 'transferencia'">📱 Transferencia</span>
            <span v-else>💳 Tarjeta</span>
          </p>
          <p v-if="s.observaciones" class="observaciones-box"><strong>Notas:</strong> {{ s.observaciones }}</p>
        </div>

        <div class="card-acciones">
          <button class="btn btn-secundario" @click="abrirModalEditar(s)">✏️ Editar</button>
          <button class="btn btn-peligro" @click="pedirConfirmacionEliminar(s)">🗑️ Eliminar</button>
        </div>
      </div>
    </main>

    <!-- Modal Formulario Mejorado -->
    <div v-if="mostrarModal" class="modal-bg" @click.self="cerrarModal">
      <div class="modal-body">
        <h2>{{ modoEdicion ? 'Editar Registro' : 'Registrar Nuevo Servicio' }}</h2>
        <form @submit.prevent="guardarServicio">
          
          <label>Nombre del Cliente:
            <input type="text" v-model="formulario.cliente" placeholder="Ej. Juan Pérez" required />
          </label>

          <!-- Selección Múltiple de Servicios con Checkboxes -->
          <fieldset class="fieldset-servicios">
            <legend>Servicios a Realizar (Seleccione uno o varios):</legend>
            <div class="checkbox-grid">
              <label v-for="cat in catalogoServicios" :key="cat.nombre" class="checkbox-label">
                <input 
                  type="checkbox" 
                  :value="cat.nombre" 
                  v-model="formulario.serviciosSeleccionados" 
                />
                {{ cat.nombre }} (${{ cat.precio.toLocaleString() }})
              </label>
            </div>
          </fieldset>

          <div class="precio-preview">
            <span>Precio Total Calculado:</span>
            <strong>${{ precioCalculado.toLocaleString() }}</strong>
          </div>

          <label>Barbero Asignado:
            <select v-model="formulario.barbero">
              <option value="Don Ramiro">Don Ramiro</option>
              <option value="Mateo">Mateo</option>
              <option value="Camilo">Camilo</option>
            </select>
          </label>

          <label>Fecha y Hora Programada:
            <input type="datetime-local" v-model="formulario.fecha" required />
          </label>

          <div class="form-row">
            <label>Método de Pago:
              <select v-model="formulario.metodoPago">
                <option value="efectivo">Efectivo</option>
                <option value="transferencia">Transferencia</option>
                <option value="tarjeta">Tarjeta</option>
              </select>
            </label>

            <label>Estado del Pago:
              <select v-model="formulario.estadoPago">
                <option value="pagado">Pagado</option>
                <option value="pendiente">Pendiente</option>
                <option value="fiado">Fiado</option>
              </select>
            </label>
          </div>

          <label>Observaciones o Notas:
            <textarea v-model="formulario.observaciones" placeholder="Detalles de la cita..."></textarea>
          </label>

          <div class="modal-btns">
            <button type="button" class="btn btn-secundario" @click="cerrarModal">Cancelar</button>
            <button type="submit" class="btn btn-primario">Guardar</button>
          </div>
        </form>
      </div>
    </div>

    <!-- Modal Confirmación Eliminar -->
    <div v-if="servicioAEliminar" class="modal-bg" @click.self="servicioAEliminar = null">
      <div class="modal-body modal-alerta">
        <h3>¿Eliminar servicio?</h3>
        <p>¿Está seguro de eliminar permanentemente el registro de <strong>{{ servicioAEliminar.cliente }}</strong>?</p>
        <div class="modal-btns">
          <button class="btn btn-secundario" @click="servicioAEliminar = null">Cancelar</button>
          <button class="btn btn-peligro" @click="borrarServicio">Sí, Eliminar</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.contenedor-dashboard {
  max-width: 1300px;
  margin: 0 auto;
  padding: 24px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background-color: #f4f6f9;
  min-height: 100vh;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: linear-gradient(135deg, #1e293b, #0f172a);
  color: white;
  padding: 24px 30px;
  border-radius: 12px;
  margin-bottom: 20px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.header h1 { margin: 0; font-size: 1.8rem; }
.header p { margin: 6px 0 0 0; color: #94a3b8; font-size: 0.95rem; }

/* Barra de métricas superior */
.metrics-bar {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 20px;
  margin-bottom: 30px;
}

.metric-card {
  background: white;
  padding: 20px 24px;
  border-radius: 12px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.02);
  border: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.metric-title {
  font-size: 0.9rem;
  color: #64748b;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.metric-value {
  font-size: 1.8rem;
  color: #1e293b;
  font-weight: bold;
}

.text-warning {
  color: #d97706 !important;
}

.section-title {
  font-size: 1.4rem;
  color: #1e293b;
  margin-bottom: 20px;
}

.vacio {
  text-align: center;
  padding: 60px;
  background: white;
  border-radius: 12px;
  color: #64748b;
  font-size: 1.1rem;
  box-shadow: 0 1px 3px rgba(0,0,0,0.05);
}

.grid-amplio {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 20px;
}

.card {
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 20px;
  background: white;
  box-shadow: 0 2px 4px rgba(0,0,0,0.02);
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.card:hover {
  transform: translateY(-3px);
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.08);
}

.card-fiado {
  border-left: 6px solid #ef4444;
  background: #fff5f5;
}

.card-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #f1f5f9;
  padding-bottom: 12px;
  margin-bottom: 12px;
}

.card-head h3 { margin: 0; font-size: 1.2rem; color: #1e293b; text-transform: capitalize; }

.badge {
  padding: 4px 10px;
  border-radius: 20px;
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
}
.badge.pagado { background: #dcfce7; color: #166534; }
.badge.pendiente { background: #fef9c3; color: #854d0e; }
.badge.fiado { background: #fee2e2; color: #991b1b; }

.card-body p { margin: 8px 0; font-size: 0.9rem; color: #475569; }
.precio-destacado { font-size: 1.1rem !important; color: #0f172a !important; font-weight: bold; }

.tag-servicio {
  display: inline-block;
  background: #e2e8f0;
  color: #334155;
  padding: 2px 8px;
  border-radius: 6px;
  font-size: 0.8rem;
  margin-right: 4px;
  margin-bottom: 4px;
}

.observaciones-box {
  background: #f8fafc;
  padding: 8px;
  border-radius: 6px;
  font-style: italic;
  font-size: 0.85rem !important;
}

.card-acciones {
  display: flex;
  gap: 10px;
  margin-top: 16px;
  border-top: 1px solid #f1f5f9;
  padding-top: 12px;
}

.btn {
  padding: 10px 16px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 600;
  font-size: 0.9rem;
  transition: background 0.2s;
}
.btn-primario { background: #d97706; color: white; }
.btn-primario:hover { background: #b45309; }

.btn-secundario { background: #e2e8f0; color: #475569; flex: 1; }
.btn-secundario:hover { background: #cbd5e1; }

.btn-peligro { background: #ef4444; color: white; flex: 1; }
.btn-peligro:hover { background: #dc2626; }

.modal-bg {
  position: fixed;
  top: 0; left: 0;
  width: 100%; height: 100%;
  background: rgba(15, 23, 42, 0.6);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-body {
  background: white;
  padding: 30px;
  border-radius: 16px;
  width: 95%;
  max-width: 550px;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
}

.modal-body h2 { margin-top: 0; color: #1e293b; font-size: 1.4rem; margin-bottom: 20px; }

.modal-body form {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.modal-body label {
  display: flex;
  flex-direction: column;
  font-size: 0.85rem;
  font-weight: 700;
  color: #334155;
  gap: 6px;
}

.modal-body input[type="text"],
.modal-body input[type="datetime-local"],
.modal-body select,
.modal-body textarea {
  padding: 10px 12px;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  font-size: 0.95rem;
  font-family: inherit;
  outline: none;
  transition: border-color 0.2s;
}

.modal-body input:focus, .modal-body select:focus, .modal-body textarea:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

.fieldset-servicios {
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  padding: 12px 16px;
  background: #f8fafc;
  font-size: 0.85rem;
  font-weight: 700;
  color: #334155;
}

.checkbox-grid {
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 10px;
  margin-top: 8px;
  display: grid;
}

.checkbox-label {
  display: flex !important;
  flex-direction: row !important;
  align-items: center;
  gap: 8px;
  font-weight: normal !important;
  cursor: pointer;
}

.precio-preview {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  padding: 12px 16px;
  border-radius: 8px;
  color: #1e40af;
  font-weight: 600;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.modal-btns {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  margin-top: 20px;
  border-top: 1px solid #f1f5f9;
  padding-top: 16px;
}
</style>