# CONTADOR DE SEGUNDOS CON DOS DISPLAYS 7 SEGMENTOS

En esta práctica de laboratorio número 3 se implementará un contador de segundos con dos 7 segmentos utilizando la FPGA de la Colorlight 8.2.
En primera instancia creamos las tablas de verdad correspondientes y la estructura de compuertas logicas basado en estas. Compararemos resultados con el diagrama RTL generado a partir del MAKEFILE. Por ultimo se sintetiza y se generan los archivos correspondientes para laimplementacion en la FPGA.

## 1.  COMPORTAMIENTO
- **DIAGRAMA DE FLUJO**
  
  <details>
     <summary>📸 Diagrama de flujo</summary>
  
     <br>
  
     ![](Diag_flujo_lab3.png)

    </details>
    

- **TABLAS DE VERDAD**
    
    <details>
     <summary>📸 Tabla de verdad unidades</summary>
  
     <br>
  
     ![](TablaVer_lab3.png)

    </details>
    

---
## 2. ESTRUCTURA
- **DIAGRAMA DE BLOQUES**
    <details>
     <summary>📸 Diagrama de Bloques</summary>
  
     <br>
  
     ![](Diag_flujo_lab3.png)

    </details>
    
- **REDES DE COMPUERTAS**
    <details>
    <summary>📸 Circuito lógica de salida</summary>
          
  <br>
          
  ![logica_sal](7_seg_salida.png)
        
    </details>

---
## 3. DISEÑO HDL
---
## RTL
<details>
  <summary>📸 Imagen RTL</summary>
  
  <br>
  
  ![RTL](RTL7seg.png)

</details>

---
## 4. SIMULACIÓN
---
## 5. IMPLEMENTACIÓN 
- **PLACE AND ROUTE**
