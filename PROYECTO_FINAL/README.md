# Velocimetro digital
<p align="justify">
Este documento describe el diseño e implementación de un velocímetro digital basado en FPGA Lattice ECP5 (Colorlight V8.2), u
tilizando un sensor magnético AS5600 para medir la velocidad de rotación y una pantalla RGB LED de 64x64 píxeles para la visualización. 
El sistema detecta la velocidad angular mediante el sensor, la convierte a km/h y la muestra en tiempo real con indicación visual de exceso 
de velocidad mediante cambio de color de fondo (verde/rojo). Se implementa un driver HUB75 para controlar la pantalla y una máquina de estados 
para la comunicación I2C con el sensor.
</p>

## Simulación
![alt text](simu.png)

## Circuito RTL
![alt text](rtl.png)

## Diagrama de caja negra
![alt text](CN.png)

## Diagrama de flujo general
![alt text](fujo.png)


<details>
<summary>Haz clic para expandir</summary>
module hall_pulse_counter (
    input  wire       clk,          // 25 MHz
    input  wire       rst_n,        
    input  wire       hall_sensor,  
    output reg  [7:0] speed_kmh     
);

    // Antirrebotes y detector de flanco descendente (Sensor Hall en PULL-UP activa en 0)
    reg [3:0] hall_sync;
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) hall_sync <= 4'b1111;
        else        hall_sync <= {hall_sync[2:0], hall_sensor};
    end
    wire hall_pulse = (hall_sync[3:2] == 2'b10);

    localparam ONE_SEC = 25_000_000;
    reg [24:0] timer_cnt;
    reg [7:0]  pulses;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            timer_cnt <= 0;
            pulses    <= 0;
            speed_kmh <= 8'd0;
        end else begin
            if (hall_pulse) begin
                pulses <= pulses + 1'b1;
            end

            if (timer_cnt >= ONE_SEC - 1) begin
                timer_cnt <= 0;
                
                // Mantiene el valor previo si el imán se detiene (pulses == 0)
                if (pulses > 0) begin
                    speed_kmh <= pulses * 8'd6; 
                    pulses    <= 0;
                end
            end else begin
                timer_cnt <= timer_cnt + 1'b1;
            end
        end
    end

endmodule
</details>
