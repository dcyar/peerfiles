<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import Peer from 'peerjs'

const currentView = ref('home') // 'home' | 'room'
const roomCode = ref('')
const joinCode = ref('')
const myPeerId = ref('')
const peer = ref(null)
const connections = ref([])
const sharedFiles = ref([])
const isHost = ref(false)
const connectedUsers = ref(0)

// Generar código aleatorio de 4 caracteres
const generateRoomCode = () => {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
  let code = ''
  for (let i = 0; i < 4; i++) {
    code += chars.charAt(Math.floor(Math.random() * chars.length))
  }
  return code
}

// Crear sala
const createRoom = () => {
  const code = generateRoomCode()
  roomCode.value = code
  myPeerId.value = code
  isHost.value = true
  
  peer.value = new Peer(code)
  
  peer.value.on('open', (id) => {
    console.log('Sala creada con ID:', id)
    currentView.value = 'room'
  })
  
  peer.value.on('connection', (conn) => {
    setupConnection(conn)
  })
  
  peer.value.on('error', (err) => {
    console.error('Error al crear sala:', err)
    alert('Error al crear la sala. El código podría estar en uso. Intenta nuevamente.')
    currentView.value = 'home'
  })
}

// Unirse a sala
const joinRoom = () => {
  if (!joinCode.value || joinCode.value.length !== 4) {
    alert('Por favor ingresa un código válido de 4 caracteres')
    return
  }
  
  const randomId = 'user_' + Math.random().toString(36).substr(2, 9)
  peer.value = new Peer(randomId)
  
  peer.value.on('open', (id) => {
    console.log('Conectado con ID:', id)
    myPeerId.value = id
    roomCode.value = joinCode.value
    
    const conn = peer.value.connect(joinCode.value)
    setupConnection(conn)
    currentView.value = 'room'
  })
  
  peer.value.on('connection', (conn) => {
    setupConnection(conn)
  })
  
  peer.value.on('error', (err) => {
    console.error('Error al unirse:', err)
    alert('Error al unirse a la sala. Verifica el código e intenta nuevamente.')
  })
}

// Configurar conexión
const setupConnection = (conn) => {
  connections.value.push(conn)
  
  conn.on('open', () => {
    console.log('Conexión establecida con:', conn.peer)
    connectedUsers.value = connections.value.length
    
    // Enviar archivos existentes al nuevo usuario
    sharedFiles.value.forEach(file => {
      conn.send({
        type: 'file',
        file: file
      })
    })
  })
  
  conn.on('data', (data) => {
    if (data.type === 'file') {
      // Verificar si el archivo ya existe
      const exists = sharedFiles.value.some(f => 
        f.name === data.file.name && f.size === data.file.size
      )
      
      if (!exists) {
        sharedFiles.value.push(data.file)
        
        // Reenviar a otros peers si es host
        if (isHost.value) {
          connections.value.forEach(c => {
            if (c.peer !== conn.peer && c.open) {
              c.send(data)
            }
          })
        }
      }
    }
  })
  
  conn.on('close', () => {
    console.log('Conexión cerrada con:', conn.peer)
    connections.value = connections.value.filter(c => c !== conn)
    connectedUsers.value = connections.value.length
  })
}

// Manejar archivos seleccionados
const handleFileSelect = (event) => {
  const files = Array.from(event.target.files)
  shareFiles(files)
}

// Manejar drag and drop
const handleDrop = (event) => {
  event.preventDefault()
  const files = Array.from(event.dataTransfer.files)
  shareFiles(files)
}

const handleDragOver = (event) => {
  event.preventDefault()
}

// Compartir archivos
const shareFiles = async (files) => {
  for (const file of files) {
    const arrayBuffer = await file.arrayBuffer()
    const fileData = {
      name: file.name,
      size: file.size,
      type: file.type,
      data: arrayBuffer,
      timestamp: Date.now()
    }
    
    sharedFiles.value.push(fileData)
    
    // Enviar a todos los peers conectados
    connections.value.forEach(conn => {
      if (conn.open) {
        conn.send({
          type: 'file',
          file: fileData
        })
      }
    })
  }
}

// Descargar archivo
const downloadFile = (fileData) => {
  const blob = new Blob([fileData.data], { type: fileData.type })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = fileData.name
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

// Formatear tamaño de archivo
const formatFileSize = (bytes) => {
  if (bytes === 0) return '0 Bytes'
  const k = 1024
  const sizes = ['Bytes', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return Math.round(bytes / Math.pow(k, i) * 100) / 100 + ' ' + sizes[i]
}

// Copiar código al portapapeles
const copyRoomCode = () => {
  navigator.clipboard.writeText(roomCode.value)
  alert('Código copiado al portapapeles')
}

// Salir de la sala
const leaveRoom = () => {
  if (peer.value) {
    peer.value.destroy()
  }
  connections.value = []
  sharedFiles.value = []
  currentView.value = 'home'
  roomCode.value = ''
  joinCode.value = ''
  isHost.value = false
  connectedUsers.value = 0
}

// Cleanup al desmontar
onUnmounted(() => {
  if (peer.value) {
    peer.value.destroy()
  }
})
</script>

<template>
  <div class="app">
    <header>
      <h1>📁 PeerFiles - Compartir Archivos P2P</h1>
    </header>

    <!-- Pantalla de inicio -->
    <div v-if="currentView === 'home'" class="home-view">
      <div class="forms-container">
        <div class="form-card">
          <h2>Crear Sala</h2>
          <p>Crea una nueva sala y comparte el código con otros usuarios</p>
          <button @click="createRoom" class="btn btn-primary">
            Crear Sala
          </button>
        </div>

        <div class="divider">O</div>

        <div class="form-card">
          <h2>Unirse a Sala</h2>
          <p>Ingresa el código de 4 caracteres para unirte a una sala existente</p>
          <input
            v-model="joinCode"
            type="text"
            placeholder="Código (4 caracteres)"
            maxlength="4"
            class="input"
            @keyup.enter="joinRoom"
          />
          <button @click="joinRoom" class="btn btn-primary">
            Unirse
          </button>
        </div>
      </div>
    </div>

    <!-- Vista de sala -->
    <div v-if="currentView === 'room'" class="room-view">
      <div class="room-header">
        <div class="room-info">
          <h2>Sala: <span class="room-code">{{ roomCode }}</span></h2>
          <button @click="copyRoomCode" class="btn btn-small">📋 Copiar Código</button>
        </div>
        <div class="room-status">
          <span>{{ isHost ? '👑 Anfitrión' : '👤 Invitado' }}</span>
          <span>{{ connectedUsers }} usuario(s) conectado(s)</span>
          <button @click="leaveRoom" class="btn btn-danger btn-small">Salir</button>
        </div>
      </div>

      <div class="room-content">
        <!-- Zona de drop -->
        <div 
          class="drop-zone"
          @drop="handleDrop"
          @dragover="handleDragOver"
        >
          <div class="drop-zone-content">
            <p class="drop-icon">📤</p>
            <p>Arrastra archivos aquí o</p>
            <label class="btn btn-secondary">
              Seleccionar Archivos
              <input
                type="file"
                multiple
                @change="handleFileSelect"
                style="display: none"
              />
            </label>
          </div>
        </div>

        <!-- Lista de archivos -->
        <div class="files-section">
          <h3>Archivos Compartidos ({{ sharedFiles.length }})</h3>
          
          <div v-if="sharedFiles.length === 0" class="empty-state">
            <p>No hay archivos compartidos aún</p>
          </div>

          <div v-else class="files-list">
            <div 
              v-for="(file, index) in sharedFiles" 
              :key="index"
              class="file-item"
            >
              <div class="file-info">
                <span class="file-icon">📄</span>
                <div class="file-details">
                  <div class="file-name">{{ file.name }}</div>
                  <div class="file-size">{{ formatFileSize(file.size) }}</div>
                </div>
              </div>
              <button @click="downloadFile(file)" class="btn btn-small btn-success">
                ⬇️ Descargar
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
}

.app {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 20px;
}

header {
  text-align: center;
  color: white;
  margin-bottom: 40px;
}

header h1 {
  font-size: 2.5rem;
  margin: 0;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.2);
}

/* Pantalla de inicio */
.home-view {
  max-width: 900px;
  margin: 0 auto;
}

.forms-container {
  display: flex;
  gap: 30px;
  align-items: center;
  justify-content: center;
  flex-wrap: wrap;
}

.form-card {
  background: white;
  padding: 30px;
  border-radius: 15px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
  flex: 1;
  min-width: 280px;
  max-width: 350px;
}

.form-card h2 {
  margin: 0 0 10px 0;
  color: #333;
}

.form-card p {
  color: #666;
  margin-bottom: 20px;
  font-size: 0.95rem;
}

.divider {
  color: white;
  font-size: 1.5rem;
  font-weight: bold;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.2);
}

.input {
  width: 100%;
  padding: 12px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  font-size: 1rem;
  margin-bottom: 15px;
  text-transform: uppercase;
  text-align: center;
  letter-spacing: 3px;
  font-weight: bold;
}

.input:focus {
  outline: none;
  border-color: #667eea;
}

.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 600;
}

.btn-primary {
  background: #667eea;
  color: white;
  width: 100%;
}

.btn-primary:hover {
  background: #5568d3;
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(102, 126, 234, 0.4);
}

.btn-secondary {
  background: #f0f0f0;
  color: #333;
}

.btn-secondary:hover {
  background: #e0e0e0;
}

.btn-success {
  background: #10b981;
  color: white;
}

.btn-success:hover {
  background: #059669;
}

.btn-danger {
  background: #ef4444;
  color: white;
}

.btn-danger:hover {
  background: #dc2626;
}

.btn-small {
  padding: 8px 16px;
  font-size: 0.9rem;
}

/* Vista de sala */
.room-view {
  max-width: 1200px;
  margin: 0 auto;
}

.room-header {
  background: white;
  padding: 20px 30px;
  border-radius: 15px;
  margin-bottom: 20px;
  box-shadow: 0 5px 20px rgba(0, 0, 0, 0.1);
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 15px;
}

.room-info {
  display: flex;
  align-items: center;
  gap: 15px;
  flex-wrap: wrap;
}

.room-info h2 {
  margin: 0;
  color: #333;
}

.room-code {
  color: #667eea;
  background: #f0f4ff;
  padding: 5px 15px;
  border-radius: 8px;
  letter-spacing: 3px;
  font-family: monospace;
}

.room-status {
  display: flex;
  align-items: center;
  gap: 15px;
  color: #666;
}

.room-content {
  display: grid;
  grid-template-columns: 1fr;
  gap: 20px;
}

/* Zona de drop */
.drop-zone {
  background: white;
  border: 3px dashed #667eea;
  border-radius: 15px;
  padding: 40px;
  text-align: center;
  transition: all 0.3s ease;
}

.drop-zone:hover {
  border-color: #5568d3;
  background: #f0f4ff;
}

.drop-zone-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
}

.drop-icon {
  font-size: 3rem;
  margin: 0;
}

/* Lista de archivos */
.files-section {
  background: white;
  border-radius: 15px;
  padding: 30px;
  box-shadow: 0 5px 20px rgba(0, 0, 0, 0.1);
}

.files-section h3 {
  margin: 0 0 20px 0;
  color: #333;
}

.empty-state {
  text-align: center;
  color: #999;
  padding: 40px;
}

.files-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.file-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px;
  background: #f9fafb;
  border-radius: 10px;
  transition: all 0.2s ease;
}

.file-item:hover {
  background: #f0f4ff;
  transform: translateX(5px);
}

.file-info {
  display: flex;
  align-items: center;
  gap: 15px;
  flex: 1;
}

.file-icon {
  font-size: 2rem;
}

.file-details {
  flex: 1;
}

.file-name {
  font-weight: 600;
  color: #333;
  margin-bottom: 3px;
  word-break: break-word;
}

.file-size {
  font-size: 0.85rem;
  color: #999;
}

@media (max-width: 768px) {
  header h1 {
    font-size: 1.8rem;
  }
  
  .forms-container {
    flex-direction: column;
  }
  
  .divider {
    transform: rotate(90deg);
  }
  
  .room-header {
    flex-direction: column;
    text-align: center;
  }
  
  .room-info,
  .room-status {
    flex-direction: column;
  }
  
  .file-item {
    flex-direction: column;
    gap: 10px;
  }
}
</style>
