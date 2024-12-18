<template>
  <div>
    <!-- Canvas para dibujar -->
    <canvas
      ref="canvas" 
      :width="canvasWidth"
      :height="canvasHeight"
      @mousedown="startDrawing"  
      @mousemove="draw"
      @mouseup="stopDrawing"
      @mouseleave="stopDrawing"
    ></canvas>
    <!-- Controles del canvas -->
     <!--
    mouseDown: Se activa cuando se presiona el boton del mouse
    mouseMove: Se activa cuando se mueve el mouse
    mouseUp: Se activa cuando se suelta el boton del mouse
    mouseLeave: Se activa cuando el mouse sale del area del canvas
     -->
    <div>
      <button @click="clearCanvas">Limpiar</button>
      <input type="color" v-model="currentColor" />
      <input type="range" v-model="lineWidth" min="1" max="20" />
    </div>
    <div class="pdf-controls">
      <input 
        type="file" 
        @change="loadPDF" 
        accept="application/pdf"   
        ref="pdfInput"
        style="margin: 10px 0;"
      />
      
      <button 
        @click="insertDrawingIntoPDF" 
        :disabled="!selectedPDF"
      >
        Insertar Dibujo en PDF y Guardar
      </button>
    </div>
  </div>
</template>

<script>
import { PDFDocument } from 'pdf-lib' //Libreria para la manipulacion de PDFs

export default {
  data() { //Funcion que retorna los datos del componente
      return {
        //Datos del componente canvas
        canvasWidth: 500,        
        canvasHeight: 300,      
        isDrawing: false,        
        currentColor: '#000000', 
        lineWidth: 5,           
        ctx: null,   //Contexto del canvas
        selectedPDF: null  //PDF seleccionado  
      }
  },

  //Mounted es un hook de ciclo de vida en canvas
  mounted() {
      this.ctx = this.$refs.canvas.getContext('2d')
  },
  methods: {
      startDrawing(event) {
          this.isDrawing = true
          this.draw(event)
      },
      
      stopDrawing() {
          this.isDrawing = false
          this.ctx.beginPath()
      },

      draw(event) {
        if (!this.isDrawing) return  // Si no estamos dibujando, no hace nada

        this.ctx.lineWidth = this.lineWidth       // Establece el grosor
        this.ctx.lineCap = 'round'                // Hace que los trazos sean redondeados
        this.ctx.strokeStyle = this.currentColor  // Establece el color

        this.ctx.lineTo(event.offsetX, event.offsetY)  // Dibuja hasta la posición que se indico
        this.ctx.stroke()                              // Realiza el trazo
        this.ctx.beginPath()                           // Inicia un nuevo trazo
        this.ctx.moveTo(event.offsetX, event.offsetY)  // Mueve el "lápiz" a la posición actual
      },
      //Se borra un rectangulo osea todo el canvas
      clearCanvas() {
          this.ctx.clearRect(0, 0, this.canvasWidth, this.canvasHeight)
      },

      loadPDF(event) {
        //Obtenemos el archivo seleccionado
          const file = event.target.files[0]
          if (file) {
              this.selectedPDF = file
          }
      },

      async insertDrawingIntoPDF() {
        if (!this.selectedPDF) {
            alert('Por favor, selecciona un PDF primero')
            return
        }

        try {
            // Crea un lector de archivos
            const fileReader = new FileReader() //FileReader es una clase/funcion de js que permite leer archivos
            
            fileReader.onload = async (e) => {
                const arrayBuffer = e.target.result
                
                // Carga el PDF seleccionado y se obtine la primera pagina
                const pdfDoc = await PDFDocument.load(arrayBuffer)
                const pages = pdfDoc.getPages()
                const firstPage = pages[0]
                
                // Convierte el dibujo a PNG
                const imgData = this.$refs.canvas.toDataURL('image/png') //Convierte el dibujo a una imagen PNG
                const imgBytes = atob(imgData.split(',')[1]) //atob decodifica una cadena de datos que ha sido codificada en base-64 a binario
                const imgArray = new Uint8Array(imgBytes.length)  //Crea un array de bytes para que se pueda insertar en el PDF
                for (let i = 0; i < imgBytes.length; i++) {
                    imgArray[i] = imgBytes.charCodeAt(i)
                }
                
                // Inserta la imagen en el PDF
                const image = await pdfDoc.embedPng(imgArray)  //embedPng inserta una imagen PNG en el PDF
                const { height } = firstPage.getSize()
                
                firstPage.drawImage(image, {
                  //Posicion y tamano de la imagen que se inserta en el PDF 
                    x: 250,                
                    y: height - 800,     
                    width: 150,          
                    height: 100,          
                })
                
                // Guarda y descarga el PDF
                const pdfBytes = await pdfDoc.save() //genero el pdf con la imagen
                const blob = new Blob([pdfBytes], { type: 'application/pdf' })
                const url = URL.createObjectURL(blob)  
                
                const link = document.createElement('a')
                link.href = url
                link.download = 'pdf-prueba-canvas.pdf'  //Nombre del archivo
                link.click()
                
                URL.revokeObjectURL(url)  // Limpia la URL temporal
            }
            
            fileReader.readAsArrayBuffer(this.selectedPDF)
        } catch (error) {
            console.error('Error al procesar el PDF:', error)
            alert('Hubo un error al procesar el PDF. Por favor, intenta nuevamente.')
        }
    }
  },
}
</script>

<style scoped>
canvas {
  border: 1px solid #000;
}
.pdf-controls {
  margin-top: 20px;
}
button {
  margin: 5px;
  padding: 8px 16px;
  cursor: pointer;
}
button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}
</style>