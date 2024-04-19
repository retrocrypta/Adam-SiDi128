//============================================================================
//  ColecoVision
//
//  Port to MiSTer
//  Copyright (C) 2017-2019 Sorgelig
//
//  This program is free software; you can redistribute it and/or modify it
//  under the terms of the GNU General Public License as published by the Free
//  Software Foundation; either version 2 of the License, or (at your option)
//  any later version.
//
//  This program is distributed in the hope that it will be useful, but WITHOUT
//  ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
//  FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License for
//  more details.
//
//  You should have received a copy of the GNU General Public License along
//  with this program; if not, write to the Free Software Foundation, Inc.,
//  51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
//============================================================================

module guest_mist
#
(
   parameter  NUM_DISKS = 2,
   parameter  NUM_TAPES = 2,
   parameter  USE_REQ   = 0,
   parameter  TOT_DISKS = NUM_DISKS + NUM_TAPES
)
(       
        output        LED,                                              
        output        VGA_HS,
        output        VGA_VS,
        output        AUDIO_L,
        output        AUDIO_R, 
		  output [15:0]  DAC_L, 
		  output [15:0]  DAC_R, 
        input         TAPE_IN,
        input         UART_RX,
        output        UART_TX,
        input         SPI_SCK,
        output        SPI_DO,
        input         SPI_DI,
        input         SPI_SS2,
        input         SPI_SS3,
		  input         SPI_SS4,
        input         CONF_DATA0,
        input         CLOCK_27,
        output  [5:0] VGA_R,
        output  [5:0] VGA_G,
        output  [5:0] VGA_B,
		  
		  output [12:0] SDRAM_A,
		  inout  [15:0] SDRAM_DQ,
		  output        SDRAM_DQML,
        output        SDRAM_DQMH,
        output        SDRAM_nWE,
        output        SDRAM_nCAS,
        output        SDRAM_nRAS,
        output        SDRAM_nCS,
        output  [1:0] SDRAM_BA,
        output        SDRAM_CLK,
        output        SDRAM_CKE
);


assign LED   = |sd_rd || ioctl_download; 

wire vga_de;
reg  en216p;

`include "build_id.v"
parameter CONF_STR = {
        "Adam;;",
        "F,COLBINROM,Load CART;",
        "F,COLBINROM,Load Ext. ROM;",
        "S0,DSK,Load Floppy 1;",
        "S1,DSK,Load Floppy 2;",
        "S2,DDP,Load Tape 1;",
        "S3,DDP,Load Tape 2;",
        "O79,Scandoubler Fx,None,HQ2x,CRT 25%,CRT 50%;",
        "O3,Joysticks swap,No,Yes;",
        "O45,RAM Size,1KB,8KB,SGM;",
        "OC,Mode,Computer,Console;",
        "T0,Reset;",
        "V,v",`BUILD_DATE
};

/////////////////  CLOCKS  ////////////////////////

wire clk_sys;
wire pll_locked;

pll pll
(
        .inclk0(CLOCK_27),
        .areset(0),
        .c0(clk_sys),
        .locked(pll_locked)
);

reg ce_10m7 = 0;
reg ce_5m3 = 0;
always @(posedge clk_sys) begin
        reg [2:0] div;

        div <= div+1'd1;
        ce_10m7 <= !div[1:0];
        ce_5m3  <= !div[2:0];
end

/////////////////  HPS  ///////////////////////////

wire [31:0] status;
wire  [1:0] buttons;

wire [31:0] joy0, joy1;

wire        ioctl_download;
wire  [7:0] ioctl_index;
wire        ioctl_wr;
wire [24:0] ioctl_addr;
wire  [7:0] ioctl_dout;
wire        forced_scandoubler;
wire [21:0] gamma_bus;

wire [31:0] sd_lba[TOT_DISKS];
wire [31:0] sd_lba_mux;
reg   [TOT_DISKS-1:0] sd_rd;
reg   [TOT_DISKS-1:0] sd_wr;
wire  [TOT_DISKS-1:0] sd_ack;
wire        sd_ack_mux;
wire  [8:0] sd_buff_addr;
wire  [7:0] sd_buff_dout;
wire  [7:0] sd_buff_din[TOT_DISKS];
wire  [7:0] sd_buff_din_mux;
wire        sd_buff_wr;

wire  [TOT_DISKS-1:0] img_mounted;
wire        img_readonly;

wire [63:0] img_size;

wire [10:0] ps2_key;
wire ypbpr;
wire [31:0] img_ext;


// multiplexing by hand the mister images.

always @(posedge clk_sys) begin
    if (sd_rd[0] || sd_wr[0]) begin
			sd_lba_mux     <= sd_lba[0];
			sd_buff_din_mux<= sd_buff_din[0];
			sd_ack[0]      <= sd_ack_mux; 
	 end
    if (sd_rd[1] || sd_wr[1]) begin
			sd_lba_mux     <= sd_lba[1];
			sd_buff_din_mux<= sd_buff_din[1];
			sd_ack[1]      <= sd_ack_mux; 
	 end
	 
	 if (sd_rd[2] || sd_wr[2]) begin
    	 sd_lba_mux     <= sd_lba[2];
		 sd_buff_din_mux<= sd_buff_din[2];
		 sd_ack[2]      <= sd_ack_mux; 
    end
	 
    if (sd_rd[3] || sd_wr[3]) begin
		 sd_lba_mux     <= sd_lba[3];
		 sd_buff_din_mux<= sd_buff_din[3];
		 sd_ack[3]      <= sd_ack_mux; 
	 end

end

user_io #(.STRLEN($size(CONF_STR)>>3),.SD_IMAGES(TOT_DISKS)) user_io
(
	.clk_sys             (clk_sys          ),
   .clk_sd              (clk_sys          ),
	.SPI_SS_IO           (CONF_DATA0),
	.SPI_CLK             (SPI_SCK),
	.SPI_MOSI            (SPI_DI),
	.SPI_MISO            (SPI_DO),

	.conf_str            (CONF_STR),
	.status              (status),
	.scandoubler_disable (forced_scandoubler),
	.ypbpr               (ypbpr),
	.no_csync            (),
	.buttons             (buttons),
	
	.ps2_key             (ps2_key),

	.sd_sdhc             (0),
	.sd_lba              (sd_lba_mux),
	.sd_rd               (sd_rd),
	.sd_wr               (sd_wr),
	.sd_ack              (sd_ack_mux),
	.sd_buff_addr        (sd_buff_addr),
	.sd_dout             (sd_buff_dout),
	.sd_din              (sd_buff_din_mux),
	.sd_dout_strobe      (sd_buff_wr),
	
	.img_mounted(img_mounted),
	.img_size(img_size),
	//.img_readonly(img_readonly),

	.joystick_0          (joy0          ),
	.joystick_1          (joy1          )
);

data_io data_io
(
	.clk_sys             (clk_sys),
	.SPI_SCK             (SPI_SCK),
	.SPI_DI              (SPI_DI),
	.SPI_SS2             (SPI_SS2),

	.clkref_n            (),
	.ioctl_fileext       (img_ext),
	.ioctl_wr            (ioctl_wr),
	.ioctl_addr          (ioctl_addr),
	.ioctl_dout          (ioctl_dout),
	.ioctl_download      (ioctl_download),
	.ioctl_index         (ioctl_index)
);



/////////////////  RESET  /////////////////////////

wire reset = !pll_locked | status[0] | buttons[1] | ioctl_download;

/////////////////  Memory  ////////////////////////

wire [12:0] bios_a;
wire  [7:0] bios_d;

rom #(.AW(13),.DW(8),.FN("../rtl/bios.hex")) rom1
(
        .clock(clk_sys),
        .address(bios_a),
        .enable(1),
        .q(bios_d)
);

wire [14:0] writer_a;
wire  [7:0] writer_d;
rom #(15,8,"../rtl/writer.hex") rom2
(
        .clock(clk_sys),
        .address(writer_a),
        .enable(1'b1),
        .q(writer_d)
);


wire [13:0] eos_a;
wire  [7:0] eos_d;

rom #(14,8,"../rtl/eos.hex") rom3
(
        .clock(clk_sys),
        .address(eos_a),
        .enable(1'b1),
        .q(eos_d)
);


wire [14:0] cpu_ram_a;
wire        ram_we_n, ram_ce_n;
wire  [7:0] ram_di;
wire  [7:0] ram_do;
wire [14:0] ram_a = cpu_ram_a;



														
  logic [15:0] ramb_addr;
  logic        ramb_wr;
  logic        ramb_rd;
  logic [7:0]  ramb_dout;
  logic        ramb_wr_ack;
  logic        ramb_rd_ack;

dpramv #(8, 15) ram
(
        .clock_a(clk_sys),
        .address_a(ram_a),
        .wren_a(ce_10m7 & ~(ram_we_n | ram_ce_n)),
        .data_a(ram_do),
        .q_a(ram_di),
        .clock_b(clk_sys),
        .address_b(ramb_addr[14:0]),
        .wren_b(ramb_wr & ~ramb_addr[15]),
        .data_b(ramb_dout),
        .q_b(),

        .enable_b(1'b1),
        .ce_a(1'b1)
);

wire [14:0]         lowerexpansion_ram_a;
wire lowerexpansion_ram_ce_n;
wire lowerexpansion_ram_rd_n;
wire lowerexpansion_ram_we_n;
wire [7:0] lowerexpansion_ram_di;
wire [7:0] lowerexpansion_ram_do;

spramv #(15) lowerexpansion_ram
    (
     .clock(clk_sys),
     .address(lowerexpansion_ram_a),
     .wren(ce_10m7 & ~(lowerexpansion_ram_we_n | lowerexpansion_ram_ce_n)),
     .data(lowerexpansion_ram_do),
     .q(lowerexpansion_ram_di),
     .cs(~lowerexpansion_ram_ce_n)
     );

wire [14:0] upper_ram_a;
wire        upper_ram_we_n, upper_ram_ce_n;
wire  [7:0] upper_ram_di;
wire  [7:0] upper_ram_do;
  dpramv #(8, 15) upper_ram
    (
     .clock_a(clk_sys),
     .address_a(upper_ram_a),
     .wren_a(ce_10m7 & ~(upper_ram_we_n | upper_ram_ce_n)),
     .data_a(upper_ram_do),
     .q_a(upper_ram_di),

     .clock_b(clk_sys),
     .address_b(ramb_addr[14:0]),
     .wren_b(ramb_wr & ramb_addr[15]),
     .data_b(ramb_dout),
     .q_b(),

     .enable_b(1'b1),
     .ce_a(1'b1)
     );

  always @(posedge clk_sys) begin
    ramb_wr_ack <= ramb_wr;
    ramb_rd_ack <= ramb_rd;
  end


wire [13:0] vram_a;
wire        vram_we;
wire  [7:0] vram_di;
wire  [7:0] vram_do;

spramv #(14) vram
(
        .clock(clk_sys),
        .address(vram_a),
        .wren(vram_we),
        .data(vram_do),
        .q(vram_di)
);

wire [19:0] cart_a;
wire  [7:0] cart_d;

spramv #(15) rom_cartridge
    (
     .clock(clk_sys),
     .address((ioctl_download && ioctl_index[4:0]==1)? ioctl_addr : cart_a),
     .wren(ioctl_wr),
     .data(ioctl_dout),
     .q(cart_d),
     .cs(1'b1)
     );

wire [19:0] ext_rom_a;
wire  [7:0] ext_rom_d;

spramv #(15) extended_rom
    (
     .clock(clk_sys),
     .address((ioctl_download && ioctl_index[4:0] == 2) ? ioctl_addr : ext_rom_a),
     .wren(ioctl_wr),
     .data(ioctl_dout),
     .q(ext_rom_d),
     .cs(1'b1)
     );
	  

////////////////  Console  ////////////////////////

wire [10:0] audio;
assign DAC_L = {audio,audio[10:5]};
assign DAC_R = {audio,audio[10:5]};

wire CLK_VIDEO = clk_sys;

wire [1:0] ctrl_p1;
wire [1:0] ctrl_p2;
wire [1:0] ctrl_p3;
wire [1:0] ctrl_p4;
wire [1:0] ctrl_p5;
wire [1:0] ctrl_p6;
wire [1:0] ctrl_p7 = 2'b11;
wire [1:0] ctrl_p8;
wire [1:0] ctrl_p9 = 2'b11;

wire [7:0] R,G,B;
wire hblank, vblank;
wire hsync, vsync;

wire [31:0] joya = status[3] ? joy1 : joy0;
wire [31:0] joyb = status[3] ? joy0 : joy1;

wire adam=1'b1;

  logic [TOT_DISKS-1:0] disk_present;
  logic [31:0]          disk_sector; // sector
  logic [TOT_DISKS-1:0] disk_load; // load the 512 byte sector
  logic [TOT_DISKS-1:0] disk_sector_loaded; // set high when sector ready
  logic [8:0]           disk_addr; // Byte to read or write from sector
  logic [TOT_DISKS-1:0] disk_wr; // Write data into sector (read when low)
  logic [TOT_DISKS-1:0] disk_flush; // sector access done, so flush (hint)
  logic [TOT_DISKS-1:0] disk_error; // out of bounds (?)
  logic [7:0]           disk_data[TOT_DISKS];
  logic [7:0]           disk_din;

cv_console
    #
    (
     .NUM_DISKS (NUM_DISKS),
     .NUM_TAPES (NUM_TAPES),
     .USE_REQ   (USE_REQ)
     )
  console
    (
        .clk_i(clk_sys),
        .clk_en_10m7_i(ce_10m7),
        .reset_n_i(~reset),
        .por_n_o(),
        .adam(adam),
        .mode(status[12]),
        .ctrl_p1_i(ctrl_p1),
        .ctrl_p2_i(ctrl_p2),
        .ctrl_p3_i(ctrl_p3),
        .ctrl_p4_i(ctrl_p4),
        .ctrl_p5_o(ctrl_p5),
        .ctrl_p6_i(ctrl_p6),
        .ctrl_p7_i(ctrl_p7),
        .ctrl_p8_o(ctrl_p8),
        .ctrl_p9_i(ctrl_p9),
        .joy0_i(~{|joya[19:6], 1'b0, joya[5:0]}),
        .joy1_i(~{|joyb[19:6], 1'b0, joyb[5:0]}),

        .bios_rom_a_o(bios_a),
        .bios_rom_d_i(bios_d),
  
        .eos_rom_a_o(eos_a),
        .eos_rom_d_i(eos_d),

        .writer_rom_a_o(writer_a),
        .writer_rom_d_i(writer_d),

        .cpu_ram_a_o(cpu_ram_a),
        .cpu_ram_we_n_o(ram_we_n),
        .cpu_ram_ce_n_o(ram_ce_n),
        .cpu_ram_d_i(ram_di),
        .cpu_ram_d_o(ram_do),

        .cpu_lowerexpansion_ram_a_o(lowerexpansion_ram_a),
        .cpu_lowerexpansion_ram_we_n_o(lowerexpansion_ram_we_n),
        .cpu_lowerexpansion_ram_ce_n_o(lowerexpansion_ram_ce_n),
        .cpu_lowerexpansion_ram_d_i(lowerexpansion_ram_di),
        .cpu_lowerexpansion_ram_d_o(lowerexpansion_ram_do),

        .cpu_upper_ram_a_o(upper_ram_a),
        .cpu_upper_ram_we_n_o(upper_ram_we_n),
        .cpu_upper_ram_ce_n_o(upper_ram_ce_n),
        .cpu_upper_ram_d_i(upper_ram_di),
        .cpu_upper_ram_d_o(upper_ram_do),

        .ramb_addr(ramb_addr),
        .ramb_wr(ramb_wr),
        .ramb_rd(ramb_rd),
        .ramb_dout(ramb_dout),
        .ramb_wr_ack(ramb_wr_ack),
        .ramb_rd_ack(ramb_rd_ack),

        .vram_a_o(vram_a),
        .vram_we_o(vram_we),
        .vram_d_o(vram_do),
        .vram_d_i(vram_di),

        .cart_a_o(cart_a),
        .cart_d_i(cart_d),
		  
		  .ext_rom_a_o(ext_rom_a),
		  .ext_rom_d_i(ext_rom_d),

        .border_i(status[6]),
        .rgb_r_o(R),
        .rgb_g_o(G),
        .rgb_b_o(B),
        .hsync_n_o(hsync),
        .vsync_n_o(vsync),
        .hblank_o(hblank),
        .vblank_o(vblank),

        .audio_o(audio),
        .disk_present(disk_present),
        .disk_sector(disk_sector),
        .disk_load(disk_load),
        .disk_sector_loaded(disk_sector_loaded),
        .disk_addr(disk_addr),
        .disk_wr(disk_wr),
        .disk_flush(disk_flush),
        .disk_error(disk_error),
        .disk_data(disk_data),
        .disk_din(disk_din),
        .ps2_key     (ps2_key)
);
  genvar                tla_i;
  generate
    for (tla_i = 0; tla_i < TOT_DISKS; tla_i++) begin : g_TL
      track_loader_adam
        #
        (
        .drive_num      (tla_i)
        )
      track_loader_a
        (
        .clk            (clk_sys),
        .reset          (reset),
        .img_mounted    (img_mounted[tla_i]),
        .img_size       (img_size),
        .lba_fdd        (sd_lba[tla_i]),
        .sd_ack         (sd_ack[tla_i]),
        .sd_rd          (sd_rd[tla_i]),
        .sd_wr          (sd_wr[tla_i]),
        .sd_buff_addr   (sd_buff_addr),
        .sd_buff_wr     (sd_buff_wr),
        .sd_buff_dout   (sd_buff_dout),
        .sd_buff_din    (sd_buff_din[tla_i]),

        // Disk interface
        .disk_present   (disk_present[tla_i]),
        .disk_sector    (disk_sector),
        .disk_load      (disk_load[tla_i]),
        .disk_sector_loaded (disk_sector_loaded[tla_i]),
        .disk_addr          (disk_addr),
        .disk_wr            (disk_wr[tla_i]),
        .disk_flush         (disk_flush[tla_i]),
        .disk_error         (disk_error[tla_i]),
        .disk_din           (disk_din),
        .disk_data          (disk_data[tla_i])
        );
    end // block: g_TL
  endgenerate





reg hs_o, vs_o;
always @(posedge CLK_VIDEO) begin
        hs_o <= ~hsync;
        if(~hs_o & ~hsync) vs_o <= ~vsync;
end

video_mixer #(.LINE_LENGTH(284), .HALF_DEPTH(0)) video_mixer
(
	.clk_sys(clk_sys),
	.ce_pix(ce_5m3),
	.ce_pix_actual(ce_5m3),
	.SPI_SCK(SPI_SCK),
	.SPI_SS3(SPI_SS3),
	.SPI_DI(SPI_DI),
	.scanlines(forced_scandoubler ? 2'b00 : {status[9:7] == 3, status[9:7] == 2}),
	.scandoubler_disable(forced_scandoubler),
	.hq2x(status[9:7]==1),
	.ypbpr(ypbpr),
	.ypbpr_full(1),
	.R(R[7:2]),
	.G(G[7:2]),
	.B(B[7:2]),
	.mono(0),
	.HSync(hs_o),
	.VSync(vs_o),
	.line_start(hblank),
	.VGA_R(VGA_R),
	.VGA_G(VGA_G),
	.VGA_B(VGA_B),
	.VGA_VS(VGA_VS),
	.VGA_HS(VGA_HS)
);

dac #(16) dac_l (
   .clk_i        (clk_sys),
   .res_n_i      (1      ),
   .dac_i        (DAC_L  ),
   .dac_o        (AUDIO_L)
);

dac #(16) dac_r (
   .clk_i        (clk_sys),
   .res_n_i      (1      ),
   .dac_i        (DAC_R  ),
   .dac_o        (AUDIO_R)
);

//////////////// Keypad emulation (by Alan Steremberg) ///////

wire       pressed = ps2_key[9];
wire [8:0] code    = ps2_key[8:0];
always @(posedge clk_sys) begin
	reg old_state;
	old_state <= ps2_key[10];

	if(old_state != ps2_key[10]) begin
		casex(code)

			'hX16: btn_1     <= pressed; // 1
			'hX1E: btn_2     <= pressed; // 2
			'hX26: btn_3     <= pressed; // 3
			'hX25: btn_4     <= pressed; // 4
			'hX2E: btn_5     <= pressed; // 5
			'hX36: btn_6     <= pressed; // 6
			'hX3D: btn_7     <= pressed; // 7
			'hX3E: btn_8     <= pressed; // 8
			'hX46: btn_9     <= pressed; // 9
			'hX45: btn_0     <= pressed; // 0

			'hX69: btn_1     <= pressed; // 1
			'hX72: btn_2     <= pressed; // 2
			'hX7A: btn_3     <= pressed; // 3
			'hX6B: btn_4     <= pressed; // 4
			'hX73: btn_5     <= pressed; // 5
			'hX74: btn_6     <= pressed; // 6
			'hX6C: btn_7     <= pressed; // 7
			'hX75: btn_8     <= pressed; // 8
			'hX7D: btn_9     <= pressed; // 9
			'hX70: btn_0     <= pressed; // 0

			'hX7C: btn_star  <= pressed; // *
			'hX59: btn_shift <= pressed; // Right Shift
			'hX12: btn_shift <= pressed; // Left Shift
			'hX7B: btn_minus <= pressed; // - on keypad


		endcase
	end
end

reg btn_1 = 0;
reg btn_2 = 0;
reg btn_3 = 0;
reg btn_4 = 0;
reg btn_5 = 0;
reg btn_6 = 0;
reg btn_7 = 0;
reg btn_8 = 0;
reg btn_9 = 0;
reg btn_0 = 0;

reg btn_star = 0;
reg btn_shift = 0;
reg btn_minus = 0;


////////////////  Control  ////////////////////////
//	"J1,dir,dir,dir,dir,Fire 1,Fire 2,*,#,[8]0,1,2,3,4,5,6,7,8,9,Purple Tr,Blue Tr;",
//        0   1   2   3   4      5     6 7 8 9 10 11 12 


wire [0:19] keypad0 = {joya[8],joya[9],joya[10],joya[11],joya[12],joya[13],joya[14],joya[15],joya[16],joya[17],joya[6],joya[7],joya[18],joya[19],joya[3],joya[2],joya[1],joya[0],joya[4],joya[5]};
wire [0:19] keypad1 = {joyb[8],joyb[9],joyb[10],joyb[11],joyb[12],joyb[13],joyb[14],joyb[15],joyb[16],joyb[17],joyb[6],joyb[7],joyb[18],joyb[19],joyb[3],joyb[2],joyb[1],joyb[0],joyb[4],joyb[5]};
wire [0:19] keyboardemu = { btn_0, btn_1, btn_2, btn_3, btn_4, btn_5, btn_6, btn_7, btn_8, btn_9, btn_star | (btn_8&btn_shift), btn_minus | (btn_shift & btn_3), 8'b0};
wire [0:19] keypad[2] = '{keypad0|keyboardemu,keypad1|keyboardemu};

reg [3:0] ctrl1[2] = '{'0,'0};
assign {ctrl_p1[0],ctrl_p2[0],ctrl_p3[0],ctrl_p4[0]} = ctrl1[0];
assign {ctrl_p1[1],ctrl_p2[1],ctrl_p3[1],ctrl_p4[1]} = ctrl1[1];

localparam cv_key_0_c        = 4'b0011;
localparam cv_key_1_c        = 4'b1110;
localparam cv_key_2_c        = 4'b1101;
localparam cv_key_3_c        = 4'b0110;
localparam cv_key_4_c        = 4'b0001;
localparam cv_key_5_c        = 4'b1001;
localparam cv_key_6_c        = 4'b0111;
localparam cv_key_7_c        = 4'b1100;
localparam cv_key_8_c        = 4'b1000;
localparam cv_key_9_c        = 4'b1011;
localparam cv_key_asterisk_c = 4'b1010;
localparam cv_key_number_c   = 4'b0101;
localparam cv_key_pt_c       = 4'b0100;
localparam cv_key_bt_c       = 4'b0010;
localparam cv_key_none_c     = 4'b1111;

generate
        genvar i;
        for (i = 0; i <= 1; i++) begin : ctl
                always_comb begin
                        reg [3:0] ctl1, ctl2;
                        reg p61,p62;

                        ctl1 = 4'b1111;
                        ctl2 = 4'b1111;
                        p61 = 1;
                        p62 = 1;

                        if (~ctrl_p5[i]) begin
                                casex(keypad[i][0:13])
                                        'b1xxxxxxxxxxxxx: ctl1 = cv_key_0_c;
                                        'b01xxxxxxxxxxxx: ctl1 = cv_key_1_c;
                                        'b001xxxxxxxxxxx: ctl1 = cv_key_2_c;
                                        'b0001xxxxxxxxxx: ctl1 = cv_key_3_c;
                                        'b00001xxxxxxxxx: ctl1 = cv_key_4_c;
                                        'b000001xxxxxxxx: ctl1 = cv_key_5_c;
                                        'b0000001xxxxxxx: ctl1 = cv_key_6_c;
                                        'b00000001xxxxxx: ctl1 = cv_key_7_c;
                                        'b000000001xxxxx: ctl1 = cv_key_8_c;
                                        'b0000000001xxxx: ctl1 = cv_key_9_c;
                                        'b00000000001xxx: ctl1 = cv_key_asterisk_c;
                                        'b000000000001xx: ctl1 = cv_key_number_c;
                                        'b0000000000001x: ctl1 = cv_key_pt_c;
                                        'b00000000000001: ctl1 = cv_key_bt_c;
                                        'b00000000000000: ctl1 = cv_key_none_c;
                                endcase
                                p61 = ~keypad[i][19]; // button 2
                        end

                        if (~ctrl_p8[i]) begin
                                ctl2 = ~keypad[i][14:17];
                                p62 = ~keypad[i][18];  // button 1
                        end

                        ctrl1[i] = ctl1 & ctl2;
                        ctrl_p6[i] = p61 & p62;
                end
        end
endgenerate


endmodule
