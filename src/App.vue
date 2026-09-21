<script setup>
import { ref } from 'vue'
import { useLocalStorage } from '@vueuse/core'

const servicios = useLocalStorage('barberia_servicios_v3', [
  {
    id: 1,
    cliente: 'Carlos Gómez',
    serviciosSeleccionados: ['Corte con máquina'],
    barbero: 'Don Ramiro',
    fecha: '2026-09-07T10:30',
    precio: 25000,
    propina: 5000,
    metodoPago: 'efectivo',
    estadoPago: 'pagado',
    observaciones: 'Cliente habitual, degradado bajo.',
    estrellas: 5,
    fotos: []
  },
  {
    id: 2,
    cliente: 'Andrés Morales',
    serviciosSeleccionados: ['Corte con tijera'],
    barbero: 'Mateo',
    fecha: '2026-09-07T15:15',
    precio: 30000,
    propina: 0,
    metodoPago: 'transferencia',
    estadoPago: 'pendiente',
    observaciones: 'Pendiente comprobante Nequi.',
    estrellas: 0,
    fotos: []
  }
])

const catalogoServicios = useLocalStorage('barberia_catalogo_v1', [
  { id: 1, nombre: 'Corte con máquina', precio: 25000 },
  { id: 2, nombre: 'Corte con tijera', precio: 30000 },
  { id: 3, nombre: 'Barba', precio: 15000 },
  { id: 4, nombre: 'Cejas', precio: 10000 },
  { id: 5, nombre: 'Tinte', precio: 70000 },
  { id: 6, nombre: 'Limpieza facial', precio: 70000 }
])

const serviciosArchivados = useLocalStorage('barberia_archivados_v1', [])

const mostrarModal = ref(false)
const modoEdicion = ref(false)
const idEdicion = ref(null)
const servicioAEliminar = ref(null)

const criterioOrden = ref('fecha-reciente')
const busquedaHistorialCliente = ref('')
const alertaFidelidad = ref('')
const errorFormulario = ref('')
const errorCatalogo = ref('')
const mostrarModalCierreCaja = ref(false)
const mostrarModalCatalogo = ref(false)

const formulario = ref({
  cliente: '',
  serviciosSeleccionados: ['Corte con máquina'],
  barbero: 'Don Ramiro',
  fecha: '',
  propina: 0,
  metodoPago: 'efectivo',
  estadoPago: 'pendiente',
  observaciones: '',
  fotos: []
})

const nuevoServicioCat = ref({ nombre: '', precio: 0 })
const servicioEditandoCat = ref(null)
const precioCalculadoModal = ref(25000)

function actualizarPrecioModal(event) {
  const servicioModificado = event ? event.target.value : null
  const estaMarcado = event ? event.target.checked : false

  if (estaMarcado) {
    if (servicioModificado === 'Corte con máquina') {
      formulario.value.serviciosSeleccionados = formulario.value.serviciosSeleccionados.filter(s => s !== 'Corte con tijera')
    } else if (servicioModificado === 'Corte con tijera') {
      formulario.value.serviciosSeleccionados = formulario.value.serviciosSeleccionados.filter(s => s !== 'Corte con máquina')
    }
  }

  let total = 0
  for (let i = 0; i < formulario.value.serviciosSeleccionados.length; i++) {
    const nombreServ = formulario.value.serviciosSeleccionados[i]
    for (let j = 0; j < catalogoServicios.value.length; j++) {
      if (catalogoServicios.value[j].nombre === nombreServ) {
        total += catalogoServicios.value[j].precio
      }
    }
  }
  precioCalculadoModal.value = total
}

function manejarSubidaFoto(event) {
  const archivos = event.target.files
  if (!archivos || archivos.length === 0) return

  for (let i = 0; i < archivos.length; i++) {
    const archivo = archivos[i]
    if (archivo.size > 1024 * 1024 * 2) {
      errorFormulario.value = `La imagen "${archivo.name}" supera el límite de 2MB.`
      continue
    }
    const lector = new FileReader()
    lector.onload = (e) => {
      formulario.value.fotos.push(e.target.result)
    }
    lector.readAsDataURL(archivo)
  }
  event.target.value = ''
}

function eliminarFoto(index) {
  formulario.value.fotos.splice(index, 1)
}

function verificarFidelidadCliente() {
  if (!formulario.value.cliente.trim()) {
    alertaFidelidad.value = ''
    return
  }
  const nombreBusq = formulario.value.cliente.trim().toLowerCase()
  const historialCliente = servicios.value.filter(s => s.cliente.trim().toLowerCase() === nombreBusq)
  if (historialCliente.length >= 4 && !modoEdicion.value) {
    alertaFidelidad.value = `¡Cliente frecuente (${historialCliente.length} visitas previas), aplica 10% de descuento!`
  } else {
    alertaFidelidad.value = ''
  }
}

function guardarItemCatalogo() {
  if (!nuevoServicioCat.value.nombre.trim() || nuevoServicioCat.value.precio <= 0) {
    errorCatalogo.value = 'Ingrese un nombre válido y un precio mayor a 0.'
    return
  }
  errorCatalogo.value = ''
  if (servicioEditandoCat.value !== null) {
    const idx = catalogoServicios.value.findIndex(s => s.id === servicioEditandoCat.value)
    if (idx !== -1) {
      catalogoServicios.value[idx].nombre = nuevoServicioCat.value.nombre
      catalogoServicios.value[idx].precio = nuevoServicioCat.value.precio
    }
    servicioEditandoCat.value = null
  } else {
    catalogoServicios.value.push({
      id: Date.now(),
      nombre: nuevoServicioCat.value.nombre,
      precio: nuevoServicioCat.value.precio
    })
  }
  nuevoServicioCat.value = { nombre: '', precio: 0 }
}

function editarItemCatalogo(item) {
  servicioEditandoCat.value = item.id
  nuevoServicioCat.value = { nombre: item.nombre, precio: item.precio }
  errorCatalogo.value = ''
}

function eliminarItemCatalogo(id) {
  catalogoServicios.value = catalogoServicios.value.filter(s => s.id !== id)
}

function obtenerEstadisticasHistorialCliente() {
  if (!busquedaHistorialCliente.value.trim()) return null
  const nombreBuscado = busquedaHistorialCliente.value.trim().toLowerCase()
  const filtrados = servicios.value.filter(s => s.cliente.trim().toLowerCase().includes(nombreBuscado))
  let totalGastado = 0
  for (let i = 0; i < filtrados.length; i++) {
    totalGastado += (filtrados[i].precio || 0) + (filtrados[i].propina || 0)
  }
  return {
    visitas: filtrados.length,
    gastado: totalGastado
  }
}

function obtenerDeudasPorCliente() {
  const deudas = {}
  for (let i = 0; i < servicios.value.length; i++) {
    const s = servicios.value[i]
    if (s.estadoPago === 'fiado') {
      const clienteNormalizado = s.cliente.trim()
      if (!deudas[clienteNormalizado]) {
        deudas[clienteNormalizado] = 0
      }
      deudas[clienteNormalizado] += (s.precio || 0) + (s.propina || 0)
    }
  }
  return deudas
}

function obtenerComisionesBarberos() {
  const comisiones = { 'Don Ramiro': 0, 'Mateo': 0, 'Camilo': 0 }
  const porcentajeComision = 0.50
  for (let i = 0; i < servicios.value.length; i++) {
    const s = servicios.value[i]
    if (comisiones[s.barbero] !== undefined) {
      comisiones[s.barbero] += (s.precio || 0) * porcentajeComision
    }
  }
  return comisiones
}

function obtenerTotalServicios() {
  return servicios.value.length
}

function obtenerVentasTotales() {
  let total = 0
  for (let i = 0; i < servicios.value.length; i++) {
    total += (servicios.value[i].precio || 0) + (servicios.value[i].propina || 0)
  }
  return total
}

function obtenerDineroPendiente() {
  let total = 0
  for (let i = 0; i < servicios.value.length; i++) {
    const s = servicios.value[i]
    if (s.estadoPago === 'pendiente' || s.estadoPago === 'fiado') {
      total += (s.precio || 0) + (s.propina || 0)
    }
  }
  return total
}

function obtenerPromedioCalificacion() {
  let suma = 0
  let count = 0
  for (let i = 0; i < servicios.value.length; i++) {
    const estrellas = servicios.value[i].estrellas || 0
    if (estrellas > 0) {
      suma += estrellas
      count++
    }
  }
  if (count === 0) return '0.0'
  return (suma / count).toFixed(1)
}

function obtenerBarberoEstrella() {
  const conteo = {}
  for (let i = 0; i < servicios.value.length; i++) {
    const b = servicios.value[i].barbero
    conteo[b] = (conteo[b] || 0) + 1
  }
  let maxBarbero = 'Ninguno'
  let maxCortes = 0
  for (const barbero in conteo) {
    if (conteo[barbero] > maxCortes) {
      maxCortes = conteo[barbero]
      maxBarbero = barbero
    }
  }
  return maxBarbero
}

function obtenerServiciosOrdenados() {
  const lista = [...servicios.value]
  lista.sort((a, b) => {
    if (criterioOrden.value === 'fecha-reciente') {
      return new Date(b.fecha) - new Date(a.fecha)
    } else if (criterioOrden.value === 'fecha-antigua') {
      return new Date(a.fecha) - new Date(b.fecha)
    } else if (criterioOrden.value === 'precio-alto') {
      return (b.precio + (b.propina || 0)) - (a.precio + (a.propina || 0))
    } else if (criterioOrden.value === 'precio-bajo') {
      return (a.precio + (a.propina || 0)) - (b.precio + (b.propina || 0))
    } else if (criterioOrden.value === 'calificacion') {
      return (b.estrellas || 0) - (a.estrellas || 0)
    }
    return 0
  })
  return lista
}

function obtenerServiciosPorTurno(nombreTurno) {
  const lista = obtenerServiciosOrdenados()
  const grupo = []
  for (let i = 0; i < lista.length; i++) {
    const s = lista[i]
    const hora = new Date(s.fecha).getHours()
    if (nombreTurno === 'Mañana' && hora >= 6 && hora < 12) {
      grupo.push(s)
    } else if (nombreTurno === 'Tarde' && hora >= 12 && hora < 19) {
      grupo.push(s)
    } else if (nombreTurno === 'Noche' && (hora >= 19 || hora < 6)) {
      grupo.push(s)
    }
  }
  return grupo
}

function realizarCierreCaja() {
  serviciosArchivados.value.push(...servicios.value)
  servicios.value = []
  mostrarModalCierreCaja.value = false
}

function obtenerResumenCierreCaja() {
  let efectivo = 0
  let transferencia = 0
  let pendientes = 0
  for (let i = 0; i < servicios.value.length; i++) {
    const s = servicios.value[i]
    const totalServicio = (s.precio || 0) + (s.propina || 0)
    if (s.estadoPago === 'pagado') {
      if (s.metodoPago === 'efectivo') efectivo += totalServicio
      else if (s.metodoPago === 'transferencia' || s.metodoPago === 'tarjeta') transferencia += totalServicio
    } else {
      pendientes += totalServicio
    }
  }
  return { efectivo, transferencia, pendientes }
}

function abrirModalCrear() {
  modoEdicion.value = false
  idEdicion.value = null
  errorFormulario.value = ''
  const ahora = new Date()
  ahora.setMinutes(ahora.getMinutes() - ahora.getTimezoneOffset())
  
  formulario.value = {
    cliente: '',
    serviciosSeleccionados: ['Corte con máquina'],
    barbero: 'Don Ramiro',
    fecha: ahora.toISOString().slice(0, 16),
    propina: 0,
    metodoPago: 'efectivo',
    estadoPago: 'pendiente',
    observaciones: '',
    fotos: []
  }
  alertaFidelidad.value = ''
  actualizarPrecioModal()
  mostrarModal.value = true
}

function abrirModalEditar(item) {
  modoEdicion.value = true
  idEdicion.value = item.id
  errorFormulario.value = ''
  const listaServicios = item.serviciosSeleccionados || [item.servicio || 'Corte con máquina']
  
  let listaFotos = item.fotos || []
  if (item.foto && listaFotos.length === 0) {
    listaFotos = [item.foto]
  }

  formulario.value = { 
    ...item, 
    serviciosSeleccionados: [...listaServicios],
    propina: item.propina || 0,
    fotos: [...listaFotos]
  }
  alertaFidelidad.value = ''
  actualizarPrecioModal()
  mostrarModal.value = true
}

function cerrarModal() {
  mostrarModal.value = false
}

function guardarServicio() {
  if (!formulario.value.cliente.trim()) {
    errorFormulario.value = 'Por favor ingrese el nombre del cliente.'
    return
  }

  if (formulario.value.serviciosSeleccionados.length === 0) {
    errorFormulario.value = 'Debe seleccionar al menos un servicio.'
    return
  }

  errorFormulario.value = ''

  let finalPrecio = precioCalculadoModal.value
  if (alertaFidelidad.value) {
    finalPrecio = finalPrecio * 0.90
  }

  const datosServicio = {
    ...formulario.value,
    precio: finalPrecio,
    propina: Number(formulario.value.propina) || 0
  }

  if (modoEdicion.value) {
    for (let i = 0; i < servicios.value.length; i++) {
      if (servicios.value[i].id === idEdicion.value) {
        const estrellasActuales = servicios.value[i].estrellas || 0
        servicios.value[i] = { ...datosServicio, id: idEdicion.value, estrellas: estrellasActuales }
        break
      }
    }
  } else {
    servicios.value.unshift({
      ...datosServicio,
      id: Date.now(),
      estrellas: 0
    })
  }
  cerrarModal()
}

function calificarServicio(id, cantidadEstrellas) {
  for (let i = 0; i < servicios.value.length; i++) {
    if (servicios.value[i].id === id) {
      servicios.value[i].estrellas = cantidadEstrellas
      break
    }
  }
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
    <header class="header">
      <div class="header-info">
        <h1>💈 Barbería Don Ramiro</h1>
        <p>Sistema de gestión avanzada</p>
      </div>
      <div class="header-acciones">
        <button class="btn btn-secundario" @click="mostrarModalCatalogo = true">⚙️ Gestionar Catálogo</button>
        <button class="btn btn-secundario" @click="mostrarModalCierreCaja = true">📥 Cierre de Caja</button>
        <button class="btn btn-primario" @click="abrirModalCrear">+ Registrar servicio</button>
      </div>
    </header>

    <section v-if="Object.keys(obtenerDeudasPorCliente()).length > 0" class="panel-deudas">
      <h3>⚠️ Recordatorios de Deudas (Fiados)</h3>
      <div class="deudas-list">
        <span v-for="(monto, cliente) in obtenerDeudasPorCliente()" :key="cliente" class="badge-deuda">
          <strong>{{ cliente }}:</strong> ${{ monto.toLocaleString() }}
        </span>
      </div>
    </section>

    <section class="metrics-bar">
      <div class="metric-card">
        <span class="metric-title">Servicios</span>
        <span class="metric-value">{{ obtenerTotalServicios() }}</span>
      </div>
      <div class="metric-card">
        <span class="metric-title">Ventas totales</span>
        <span class="metric-value">${{ obtenerVentasTotales().toLocaleString() }}</span>
      </div>
      <div class="metric-card">
        <span class="metric-title">Dinero pendiente</span>
        <span class="metric-value text-warning">${{ obtenerDineroPendiente().toLocaleString() }}</span>
      </div>
      <div class="metric-card">
        <span class="metric-title">Promedio calificación</span>
        <span class="metric-value">⭐ {{ obtenerPromedioCalificacion() }}</span>
      </div>
      <div class="metric-card">
        <span class="metric-title">Barbero estrella</span>
        <span class="metric-value barbero-top">✂️ {{ obtenerBarberoEstrella() }}</span>
      </div>
    </section>

    <section class="toolbar-section">
      <div class="historial-busqueda">
        <label>🔍 Historial de cliente:
          <input type="text" v-model="busquedaHistorialCliente" placeholder="Escriba nombre del cliente..." />
        </label>
        <div v-if="obtenerEstadisticasHistorialCliente()" class="resultado-historial">
          <span>Visitas: <strong>{{ obtenerEstadisticasHistorialCliente().visitas }}</strong></span>
          <span>Gastado total: <strong>${{ obtenerEstadisticasHistorialCliente().gastado.toLocaleString() }}</strong></span>
        </div>
      </div>

      <div class="ordenamiento-box">
        <label>Ordenar servicios por:
          <select v-model="criterioOrden">
            <option value="fecha-reciente">Más recientes</option>
            <option value="fecha-antigua">Más antiguos</option>
            <option value="precio-alto">Mayor precio</option>
            <option value="precio-bajo">Menor precio</option>
            <option value="calificacion">Mayor calificación</option>
          </select>
        </label>
      </div>
    </section>

    <section class="comisiones-panel">
      <h4>💼 Comisiones totales (50% de servicios)</h4>
      <div class="comisiones-grid">
        <div v-for="(comision, barb) in obtenerComisionesBarberos()" :key="barb" class="comision-card">
          <span>{{ barb }}</span>
          <strong>${{ comision.toLocaleString() }}</strong>
        </div>
      </div>
    </section>

    <h2 class="section-title">Servicios registrados</h2>

    <div v-if="servicios.length === 0" class="vacio">
      <p>No hay servicios registrados en este momento.</p>
    </div>

    <div v-else>
      <div v-for="nombreTurno in ['Mañana', 'Tarde', 'Noche']" :key="nombreTurno">
        <div v-if="obtenerServiciosPorTurno(nombreTurno).length > 0">
          <div class="separador-turno">
            <span>☀️ Turno {{ nombreTurno }} ({{ obtenerServiciosPorTurno(nombreTurno).length }})</span>
          </div>

          <main class="grid-amplio">
            <div 
              v-for="s in obtenerServiciosPorTurno(nombreTurno)" 
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

              <div v-if="s.fotos && s.fotos.length > 0" class="card-fotos-grid">
                <div v-for="(img, idx) in s.fotos" :key="idx" class="card-foto">
                  <img :src="img" alt="Foto servicio" />
                </div>
              </div>
              <div v-else-if="s.foto" class="card-foto">
                <img :src="s.foto" alt="Foto servicio" />
              </div>

              <div class="card-body">
                <p><strong>Servicios:</strong> 
                  <span class="tag-servicio" v-for="(serv, idx) in (s.serviciosSeleccionados || [s.servicio])" :key="idx">
                    {{ serv }}
                  </span>
                </p>
                <p><strong>Barbero:</strong> ✂️ {{ s.barbero }}</p>
                <p><strong>Fecha y Hora:</strong> 📅 {{ new Date(s.fecha).toLocaleString() }}</p>
                
                <p class="precio-destacado">
                  <strong>Total:</strong> ${{ s.precio.toLocaleString() }}
                  <span v-if="s.propina > 0" class="propina-texto"> + ${{ s.propina.toLocaleString() }} propina</span>
                </p>

                <p>
                  <strong>Método de Pago:</strong> 
                  <span v-if="s.metodoPago === 'efectivo'">💵 Efectivo</span>
                  <span v-else-if="s.metodoPago === 'transferencia'">📱 Transferencia</span>
                  <span v-else>💳 Tarjeta</span>
                </p>
                <p v-if="s.observaciones" class="observaciones-box"><strong>Notas:</strong> {{ s.observaciones }}</p>

                <div class="rating-section">
                  <span class="rating-label">Calificación del servicio:</span>
                  <div class="estrellas-container">
                    <button 
                      v-for="n in 5" 
                      :key="n" 
                      type="button" 
                      class="btn-estrella" 
                      :class="{ 'activa': n <= (s.estrellas || 0) }"
                      @click="calificarServicio(s.id, n)"
                      :title="`Calificar con ${n} estrellas`"
                    >
                      ★
                    </button>
                  </div>
                </div>
              </div>

              <div class="card-acciones">
                <button class="btn btn-secundario" @click="abrirModalEditar(s)">✏️ Editar</button>
                <button class="btn btn-peligro" @click="pedirConfirmacionEliminar(s)">🗑️ Eliminar</button>
              </div>
            </div>
          </main>
        </div>
      </div>
    </div>

    <!-- Modal Formulario -->
    <div v-if="mostrarModal" class="modal-bg" @click.self="cerrarModal">
      <div class="modal-body">
        <h2>{{ modoEdicion ? 'Editar Registro' : 'Registrar Nuevo Servicio' }}</h2>
        
        <div v-if="errorFormulario" class="alerta-error">
          ⚠️ {{ errorFormulario }}
        </div>

        <div v-if="alertaFidelidad" class="alerta-fidelidad">
          🎉 {{ alertaFidelidad }}
        </div>

        <form @submit.prevent="guardarServicio">
          
          <label>Nombre del Cliente:
            <input type="text" v-model="formulario.cliente" @input="verificarFidelidadCliente" placeholder="Ej. Juan Pérez" autocomplete="off" />
          </label>

          <fieldset class="fieldset-servicios">
            <legend>Servicios a Realizar (Seleccione uno o varios):</legend>
            <div class="checkbox-grid">
              <label v-for="cat in catalogoServicios" :key="cat.id" class="checkbox-label">
                <input 
                  type="checkbox" 
                  :value="cat.nombre" 
                  v-model="formulario.serviciosSeleccionados" 
                  @change="actualizarPrecioModal"
                />
                {{ cat.nombre }} (${{ cat.precio.toLocaleString() }})
              </label>
            </div>
          </fieldset>

          <div class="precio-preview">
            <span>Precio Total Calculado:</span>
            <strong>${{ precioCalculadoModal.toLocaleString() }}</strong>
          </div>

          <label>Propina Opcional:
            <input type="number" v-model="formulario.propina" min="0" step="1000" placeholder="Ej. 5000" />
          </label>

          <label>Barbero Asignado:
            <select v-model="formulario.barbero">
              <option value="Don Ramiro">Don Ramiro</option>
              <option value="Mateo">Mateo</option>
              <option value="Camilo">Camilo</option>
            </select>
          </label>

          <label>Fecha y Hora Programada:
            <input type="datetime-local" v-model="formulario.fecha" />
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

          <label>Fotos del Resultado (Puedes seleccionar varias):
            <input type="file" accept="image/*" multiple @change="manejarSubidaFoto" />
          </label>
          
          <div v-if="formulario.fotos && formulario.fotos.length > 0" class="preview-fotos-list">
            <div v-for="(img, idx) in formulario.fotos" :key="idx" class="preview-foto-container">
              <img :src="img" alt="Preview" />
              <button type="button" class="btn btn-peligro btn-sm" @click="eliminarFoto(idx)">Quitar</button>
            </div>
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

    <!-- Modal Catálogo Editable -->
    <div v-if="mostrarModalCatalogo" class="modal-bg" @click.self="mostrarModalCatalogo = false">
      <div class="modal-body">
        <h2>⚙️ Gestión del Catálogo de Servicios</h2>

        <div v-if="errorCatalogo" class="alerta-error">
          ⚠️ {{ errorCatalogo }}
        </div>

        <div class="catalogo-form-container">
          <input type="text" v-model="nuevoServicioCat.nombre" placeholder="Nombre del servicio" />
          <input type="number" v-model="nuevoServicioCat.precio" placeholder="Precio base" />
          <button class="btn btn-primario" @click="guardarItemCatalogo">{{ servicioEditandoCat !== null ? 'Actualizar' : 'Agregar' }}</button>
        </div>
        <ul class="catalogo-list">
          <li v-for="item in catalogoServicios" :key="item.id">
            <span>{{ item.nombre }} - <strong>${{ item.precio.toLocaleString() }}</strong></span>
            <div class="catalogo-acciones">
              <button class="btn btn-secundario btn-sm" @click="editarItemCatalogo(item)">Editar</button>
              <button class="btn btn-peligro btn-sm" @click="eliminarItemCatalogo(item.id)">Borrar</button>
            </div>
          </li>
        </ul>
        <div class="modal-btns">
          <button class="btn btn-secundario" @click="mostrarModalCatalogo = false">Cerrar</button>
        </div>
      </div>
    </div>

    <!-- Modal Cierre de Caja -->
    <div v-if="mostrarModalCierreCaja" class="modal-bg" @click.self="mostrarModalCierreCaja = false">
      <div class="modal-body">
        <h2>📥 Resumen de Cierre de Caja</h2>
        <div class="cierre-resumen-box">
          <p>Total en Efectivo: <strong>${{ obtenerResumenCierreCaja().efectivo.toLocaleString() }}</strong></p>
          <p>Total en Transferencia/Tarjeta: <strong>${{ obtenerResumenCierreCaja().transferencia.toLocaleString() }}</strong></p>
          <p>Pendientes por Cobrar: <strong class="text-warning">${{ obtenerResumenCierreCaja().pendientes.toLocaleString() }}</strong></p>
        </div>
        <p class="cierre-advertencia">Al realizar el cierre, los servicios actuales se archivarán y la vista principal quedará limpia para un nuevo día.</p>
        <div class="modal-btns">
          <button class="btn btn-secundario" @click="mostrarModalCierreCaja = false">Cancelar</button>
          <button class="btn btn-peligro" @click="realizarCierreCaja">Confirmar Cierre y Archivar</button>
        </div>
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
  max-width: 1800px;
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
  flex-wrap: wrap;
  gap: 15px;
}

.header-acciones {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.header h1 { margin: 0; font-size: 1.8rem; }
.header p { margin: 6px 0 0 0; color: #94a3b8; font-size: 0.95rem; }

.panel-deudas {
  background: #fef2f2;
  border: 1px solid #fecaca;
  padding: 16px 20px;
  border-radius: 12px;
  margin-bottom: 20px;
}
.panel-deudas h3 { margin: 0 0 10px 0; color: #991b1b; font-size: 1.1rem; }
.deudas-list {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}
.badge-deuda {
  background: #fee2e2;
  color: #991b1b;
  padding: 6px 12px;
  border-radius: 8px;
  font-size: 0.9rem;
}

.metrics-bar {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 20px;
  margin-bottom: 20px;
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
  font-size: 0.85rem;
  color: #64748b;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.metric-value {
  font-size: 1.6rem;
  color: #1e293b;
  font-weight: bold;
}

.text-warning {
  color: #d97706 !important;
}

.barbero-top {
  font-size: 1.3rem !important;
  color: #0284c7 !important;
}

.toolbar-section {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  background: white;
  padding: 16px 20px;
  border-radius: 12px;
  border: 1px solid #e2e8f0;
  margin-bottom: 20px;
  flex-wrap: wrap;
  gap: 15px;
}

.historial-busqueda, .ordenamiento-box {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 0.85rem;
  font-weight: 700;
  color: #334155;
}

.historial-busqueda input, .ordenamiento-box select {
  padding: 8px 12px;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  font-size: 0.9rem;
  font-family: inherit;
  outline: none;
}

.resultado-historial {
  display: flex;
  gap: 15px;
  font-size: 0.85rem;
  color: #0f172a;
  background: #f8fafc;
  padding: 6px 10px;
  border-radius: 6px;
  border: 1px solid #e2e8f0;
}

.comisiones-panel {
  background: white;
  padding: 16px 20px;
  border-radius: 12px;
  border: 1px solid #e2e8f0;
  margin-bottom: 25px;
}
.comisiones-panel h4 { margin: 0 0 12px 0; color: #1e293b; font-size: 1.05rem; }
.comisiones-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 15px;
}
.comision-card {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  padding: 12px;
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 0.9rem;
  color: #475569;
}
.comision-card strong { font-size: 1.2rem; color: #0f172a; }

.section-title {
  font-size: 1.4rem;
  color: #1e293b;
  margin-bottom: 20px;
}

.separador-turno {
  margin: 25px 0 15px 0;
  font-size: 1.1rem;
  font-weight: bold;
  color: #334155;
  border-bottom: 2px solid #cbd5e1;
  padding-bottom: 6px;
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

.card-fotos-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
  gap: 8px;
  margin-bottom: 12px;
}

.card-foto {
  width: 100%;
  height: 120px;
  overflow: hidden;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
}
.card-foto img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

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
.propina-texto { font-size: 0.85rem; color: #16a34a; font-weight: 600; }

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

.rating-section {
  margin-top: 14px;
  padding-top: 10px;
  border-top: 1px dashed #e2e8f0;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.rating-label {
  font-size: 0.8rem;
  font-weight: 700;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.3px;
}

.estrellas-container {
  display: flex;
  gap: 4px;
}

.btn-estrella {
  background: transparent;
  border: none;
  font-size: 1.4rem;
  color: #cbd5e1;
  cursor: pointer;
  padding: 0;
  transition: transform 0.1s ease, color 0.2s ease;
}

.btn-estrella:hover {
  transform: scale(1.2);
}

.btn-estrella.activa {
  color: #f59e0b;
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
.btn-sm { padding: 6px 10px; font-size: 0.8rem; }
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
.modal-body input[type="number"],
.modal-body input[type="datetime-local"],
.modal-body input[type="file"],
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

.alerta-fidelidad {
  background: #ecfdf5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 12px;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.9rem;
  margin-bottom: 15px;
}

.alerta-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 12px;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.9rem;
  margin-bottom: 15px;
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

.preview-fotos-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: 150px;
  overflow-y: auto;
}

.preview-foto-container {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #f8fafc;
  padding: 6px 10px;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
}
.preview-foto-container img {
  width: 50px;
  height: 50px;
  object-fit: cover;
  border-radius: 6px;
}

.catalogo-form-container {
  display: flex;
  gap: 10px;
  margin-bottom: 15px;
}
.catalogo-form-container input {
  flex: 1;
  padding: 8px;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
}
.catalogo-list {
  list-style: none;
  padding: 0;
  margin: 0 0 20px 0;
  max-height: 250px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.catalogo-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #f8fafc;
  padding: 8px 12px;
  border-radius: 6px;
  border: 1px solid #e2e8f0;
  font-size: 0.9rem;
}
.catalogo-acciones {
  display: flex;
  gap: 6px;
}

.cierre-resumen-box {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  padding: 16px;
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 15px;
  font-size: 1rem;
}
.cierre-advertencia {
  font-size: 0.85rem;
  color: #64748b;
  font-style: italic;
  margin-bottom: 20px;
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