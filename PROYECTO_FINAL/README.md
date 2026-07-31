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

## Código en verilog
<details>
<summary>top_module.v</summary>

module top_module (
    input  wire       clk,      // 25 MHz
    input  wire       rst_n,

    // Sensor I2C (Conector J1)
    output wire       scl,
    inout  wire       sda,

    // Matriz HUB75 (Conector J16)
    output wire [2:0] rgb1,
    output wire [2:0] rgb2,
    output wire [4:0] row_addr,
    output wire       pclk,
    output wire       lat,
    output wire       oe
);

    wire [11:0] angle;
    wire        angle_valid;
    wire [3:0]  digit_tens;
    wire [3:0]  digit_units;
    wire        over_limit;

    // Instancia Lector AS5600
    as5600_speed_reader u_as5600 (
        .clk(clk),
        .rst_n(rst_n),
        .scl(scl),
        .sda(sda),
        .angle(angle),
        .valid(angle_valid)
    );

    // Instancia Calculador de Velocidad
    speed_calc u_speed (
        .clk(clk),
        .rst_n(rst_n),
        .angle(angle),
        .angle_valid(angle_valid),
        .digit_tens(digit_tens),
        .digit_units(digit_units),
        .over_limit(over_limit)
    );

    // Instancia Controlador Pantalla RGB HUB75
    hub75_driver u_display (
        .clk(clk),
        .rst_n(rst_n),
        .digit_tens(digit_tens),
        .digit_units(digit_units),
        .over_limit(over_limit),
        .rgb1(rgb1),
        .rgb2(rgb2),
        .row_addr(row_addr),
        .pclk(pclk),
        .lat(lat),
        .oe(oe)
    );

endmodule
</details>


<details>
<summary>Hall_pulse_counter</summary>
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
<details>
<summary>hub75_driver.v</summary>

module hub75_driver #(
    parameter MATRIX_WIDTH  = 64,
    parameter MATRIX_HEIGHT = 64
)(
    input  wire       clk,          // Reloj 25 MHz
    input  wire       rst_n,
    input  wire [3:0] digit_tens,
    input  wire [3:0] digit_units,
    input  wire       over_limit,   // 0: Verde, 1: Rojo (Exceso)

    output reg  [2:0] rgb1,
    output reg  [2:0] rgb2,
    output reg  [4:0] row_addr,
    output reg        pclk,
    output reg        lat,
    output reg        oe
);

    reg [1:0] clk_div;
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) clk_div <= 2'd0;
        else        clk_div <= clk_div + 1'b1;
    end
    wire tick = (clk_div == 2'd3);

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) pclk <= 1'b0;
        else if (tick) pclk <= ~pclk;
    end

    reg [5:0] x_cnt;
    reg [4:0] y_cnt;

    wire pclk_negedge = tick && (pclk == 1'b1);
    wire pclk_posedge = tick && (pclk == 1'b0);

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            x_cnt <= 6'd0;
            y_cnt <= 5'd0;
        end else if (pclk_negedge) begin
            if (x_cnt == 6'd63) begin
                x_cnt <= 6'd0;
                y_cnt <= y_cnt + 1'b1;
            end else begin
                x_cnt <= x_cnt + 1'b1;
            end
        end
    end

    // Control Latch y OE
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            lat      <= 1'b0;
            oe       <= 1'b1;
            row_addr <= 5'd0;
        end else if (pclk_posedge) begin
            if (x_cnt == 6'd60) begin
                oe <= 1'b1;
            end else if (x_cnt == 6'd62) begin
                row_addr <= y_cnt;
                lat <= 1'b1;
            end else if (x_cnt == 6'd63) begin
                lat <= 1'b0;
            end else if (x_cnt == 6'd2) begin
                oe  <= 1'b0;
            end
        end
    end

    wire [5:0] y_abs_top = {1'b0, y_cnt};
    wire [5:0] y_abs_bot = {1'b1, y_cnt};

    function draw_digit(input [3:0] digit, input [5:0] y, input [5:0] x_rel);
        reg [4:0] ry;
        begin
            draw_digit = 1'b0;
            if (y >= 6'd18 && y <= 6'd45) begin
                ry = y - 6'd18;
                case (digit)
                    4'd0: if (ry <= 3 || ry >= 24 || x_rel <= 2 || x_rel >= 13) draw_digit = 1'b1;
                    4'd1: if (x_rel >= 6 && x_rel <= 9) draw_digit = 1'b1;
                    4'd2: if (ry <= 3 || ry >= 24 || (ry >= 12 && ry <= 15) || (ry < 13 && x_rel >= 13) || (ry > 13 && x_rel <= 2)) draw_digit = 1'b1;
                    4'd3: if (ry <= 3 || ry >= 24 || (ry >= 12 && ry <= 15) || x_rel >= 13) draw_digit = 1'b1;
                    4'd4: if ((ry >= 12 && ry <= 15) || x_rel >= 13 || (ry < 13 && x_rel <= 2)) draw_digit = 1'b1;
                    4'd5: if (ry <= 3 || ry >= 24 || (ry >= 12 && ry <= 15) || (ry < 13 && x_rel <= 2) || (ry > 13 && x_rel >= 13)) draw_digit = 1'b1;
                    4'd6: if (ry <= 3 || ry >= 24 || (ry >= 12 && ry <= 15) || x_rel <= 2 || (ry > 13 && x_rel >= 13)) draw_digit = 1'b1;
                    4'd7: if (ry <= 3 || x_rel >= 13) draw_digit = 1'b1;
                    4'd8: if (ry <= 3 || ry >= 24 || (ry >= 12 && ry <= 15) || x_rel <= 2 || x_rel >= 13) draw_digit = 1'b1;
                    4'd9: if (ry <= 3 || ry >= 24 || (ry >= 12 && ry <= 15) || x_rel >= 13 || (ry < 13 && x_rel <= 2)) draw_digit = 1'b1;
                    default: draw_digit = 1'b0;
                endcase
            end
        end
    endfunction

    // Evaluaciones de píxeles
    wire pix_tens_top  = (x_cnt >= 6'd12 && x_cnt <= 6'd27) ? draw_digit(digit_tens,  y_abs_top, x_cnt - 6'd12) : 1'b0;
    wire pix_units_top = (x_cnt >= 6'd36 && x_cnt <= 6'd51) ? draw_digit(digit_units, y_abs_top, x_cnt - 6'd36) : 1'b0;
    wire is_fg_top     = pix_tens_top | pix_units_top;

    wire pix_tens_bot  = (x_cnt >= 6'd12 && x_cnt <= 6'd27) ? draw_digit(digit_tens,  y_abs_bot, x_cnt - 6'd12) : 1'b0;
    wire pix_units_bot = (x_cnt >= 6'd36 && x_cnt <= 6'd51) ? draw_digit(digit_units, y_abs_bot, x_cnt - 6'd36) : 1'b0;
    wire is_fg_bot     = pix_tens_bot | pix_units_bot;


    wire [2:0] bg_color = over_limit ? 3'b100 : 3'b010;
    wire [2:0] fg_color = 3'b000;

    always @(*) begin
        rgb1 = is_fg_top ? fg_color : bg_color;
        rgb2 = is_fg_bot ? fg_color : bg_color;
    end

endmodule

</details>

<details>
<summary>spee_calc.v</summary>

module speed_calc (
    input  wire        clk,          // Reloj 25 MHz
    input  wire        rst_n,
    input  wire [11:0] angle,
    input  wire        angle_valid,

    output reg  [3:0]  digit_tens,
    output reg  [3:0]  digit_units,
    output reg         over_limit
);

    reg [23:0] timer;
    reg [11:0] prev_angle;
    reg [11:0] delta_angle;
    reg [7:0]  speed_kmh;

    // Muestreo cada ~100ms (2,500,000 ciclos a 25 MHz)
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            timer      <= 24'd0;
            prev_angle <= 12'd0;
            speed_kmh  <= 8'd0;
        end else begin
            if (timer >= 24'd2500000) begin
                timer <= 24'd0;
                if (angle >= prev_angle)
                    delta_angle <= angle - prev_angle;
                else
                    delta_angle <= (12'd4095 - prev_angle) + angle;

                prev_angle <= angle;

                // Escalamiento simplificado para obtener Km/h
                speed_kmh <= (delta_angle * 8'd15) >> 7;
            end else begin
                timer <= timer + 1'b1;
            end
        end
    end

    // BCD y Verificación de Límite (10 km/h)
    always @(*) begin
        digit_tens  = (speed_kmh / 10) % 10;
        digit_units = speed_kmh % 10;
        over_limit  = (speed_kmh > 8'd10);
    end

endmodule

</details>
<details>
<summary>top_module.lpf</summary>

# Reloj y Botón Reset
LOCATE COMP "clk" SITE "P6";
IOBUF PORT "clk" IO_TYPE=LVCMOS33;

LOCATE COMP "rst_n" SITE "R7";
IOBUF PORT "rst_n" IO_TYPE=LVCMOS33;

# Sensor AS5600 (Conector J1)
LOCATE COMP "scl" SITE "C4";
IOBUF PORT "scl" IO_TYPE=LVCMOS33;

LOCATE COMP "sda" SITE "D4";
IOBUF PORT "sda" IO_TYPE=LVCMOS33;

# Matriz HUB75 (Conector J16)
LOCATE COMP "rgb1[0]" SITE "G14"; # R1
LOCATE COMP "rgb1[1]" SITE "G13"; # G1
LOCATE COMP "rgb1[2]" SITE "F12"; # B1

LOCATE COMP "rgb2[0]" SITE "F13"; # R2
LOCATE COMP "rgb2[1]" SITE "F14"; # G2
LOCATE COMP "rgb2[2]" SITE "H12"; # B2

IOBUF PORT "rgb1[0]" IO_TYPE=LVCMOS33;
IOBUF PORT "rgb1[1]" IO_TYPE=LVCMOS33;
IOBUF PORT "rgb1[2]" IO_TYPE=LVCMOS33;
IOBUF PORT "rgb2[0]" IO_TYPE=LVCMOS33;
IOBUF PORT "rgb2[1]" IO_TYPE=LVCMOS33;
IOBUF PORT "rgb2[2]" IO_TYPE=LVCMOS33;

LOCATE COMP "row_addr[0]" SITE "N4";  # A
LOCATE COMP "row_addr[1]" SITE "N5";  # B
LOCATE COMP "row_addr[2]" SITE "N3";  # C
LOCATE COMP "row_addr[3]" SITE "P3";  # D
LOCATE COMP "row_addr[4]" SITE "E14"; # E

IOBUF PORT "row_addr[0]" IO_TYPE=LVCMOS33;
IOBUF PORT "row_addr[1]" IO_TYPE=LVCMOS33;
IOBUF PORT "row_addr[2]" IO_TYPE=LVCMOS33;
IOBUF PORT "row_addr[3]" IO_TYPE=LVCMOS33;
IOBUF PORT "row_addr[4]" IO_TYPE=LVCMOS33;

LOCATE COMP "pclk" SITE "M3"; # Clock HUB75
LOCATE COMP "lat"  SITE "N1"; # Latch
LOCATE COMP "oe"   SITE "M4"; # Output Enable

IOBUF PORT "pclk" IO_TYPE=LVCMOS33;
IOBUF PORT "lat"  IO_TYPE=LVCMOS33;
IOBUF PORT "oe"   IO_TYPE=LVCMOS33;

</details>
<details>
<summary>Testbench</summary>
`timescale 1ns / 1ps

module tb_top;

    // Señales del Testbench
    reg        clk;
    reg        rst_n;

    wire       sda;
    wire       scl;

    wire [2:0] rgb1;
    wire [2:0] rgb2;
    wire [4:0] row_addr;
    wire       pclk;
    wire       lat;
    wire       oe;

    // Pull-up pasivo para I2C
    assign (pull1, pull0) sda = 1'b1;

    // Instancia del Módulo Principal
    top_module uut (
        .clk      (clk),
        .rst_n    (rst_n),
        .sda      (sda),
        .scl      (scl),
        .rgb1     (rgb1),
        .rgb2     (rgb2),
        .row_addr (row_addr),
        .pclk     (pclk),
        .lat      (lat),
        .oe       (oe)
    );

    // Reloj a 25 MHz (T = 40 ns -> invierte cada 20 ns)
    always #20 clk = ~clk;

    initial begin
        $dumpfile("tb_top.vcd");
        $dumpvars(0, tb_top);

        clk   = 1'b0;
        rst_n = 1'b0;

        $display("=========================================================");
        $display("   INICIANDO SIMULACIÓN: VELOCÍMETRO AS5600 + HUB75      ");
        $display("=========================================================");

        #100;
        rst_n = 1'b1;
        $display("[%0t ns] Reset liberado.", $time);

        // Simular por más tiempo para ver cambios
        #20000000;

        $display("[%0t ns] Simulación finalizada correctamente.", $time);
        $finish;
    end

    always @(posedge lat) begin
        $display("[%0t ns] [HUB75] Latch de línea -> Fila actual: %d", $time, row_addr);
    end

endmodule


</details>



