// src/project.v
`default_nettype none

module tt_um_vga_example (
    input  wire [7:0] ui_in,
    output wire [7:0] uo_out,
    input  wire [7:0] uio_in,
    output wire [7:0] uio_out,
    output wire [7:0] uio_oe,
    input  wire       ena,
    input  wire       clk,
    input  wire       rst_n
);

    wire       hsync;
    wire       vsync;
    wire       display_on;
    wire [9:0] pix_x;
    wire [9:0] pix_y;

    hvsync_generator hvsync_gen (
        .clk(clk),
        .reset(~rst_n),
        .hsync(hsync),
        .vsync(vsync),
        .display_on(display_on),
        .hpos(pix_x),
        .vpos(pix_y)
    );

    wire blink_en  = ui_in[0];
    wire allseg_en = ui_in[1];

    localparam [1:0] S1 = 2'd0;
    localparam [1:0] S2 = 2'd1;
    localparam [1:0] S3 = 2'd2;

    localparam [21:0] CLOCKS_PER_SECOND_MINUS_1 = 22'd2_499_999;

    reg [21:0] second_divider;
    reg        tick;

    always @(posedge clk) begin
        if (!rst_n) begin
            second_divider <= 22'd0;
            tick    <= 1'b0;
        end else if (second_divider == CLOCKS_PER_SECOND_MINUS_1) begin
            second_divider <= 22'd0;
            tick    <= 1'b1;
        end else begin
            second_divider <= second_divider + 22'd1;
            tick    <= 1'b0;
        end
    end

    reg [1:0] state;
    reg [4:0] state_seconds;

    always @(posedge clk) begin
        if (!rst_n) begin
            state         <= S1;
            state_seconds <= 5'd0;
        end else if (tick) begin
            case (state)
                S1: begin
                    if (state_seconds == 5'd9) begin
                        state         <= S2;
                        state_seconds <= 5'd0;
                    end else begin
                        state_seconds <= state_seconds + 5'd1;
                    end
                end

                S2: begin
                    if (state_seconds == 5'd19) begin
                        state         <= S3;
                        state_seconds <= 5'd0;
                    end else begin
                        state_seconds <= state_seconds + 5'd1;
                    end
                end

                S3: begin
                    if (state_seconds == 5'd9) begin
                        state         <= S1;
                        state_seconds <= 5'd0;
                    end else begin
                        state_seconds <= state_seconds + 5'd1;
                    end
                end

                default: begin
                    state         <= S1;
                    state_seconds <= 5'd0;
                end
            endcase
        end
    end

    reg blink_phase;
    reg blink_en_d;

    always @(posedge clk) begin
        if (!rst_n) begin
            blink_phase <= 1'b0;
            blink_en_d  <= 1'b0;
        end else begin
            blink_en_d <= blink_en;

            if (blink_en && !blink_en_d)
                blink_phase <= 1'b1; 
            else if (tick)
                blink_phase <= ~blink_phase;
        end
    end

    wire normal_segment_on =
        (state == S2) ? state_seconds[0] : 1'b1;

    wire [6:0] digit_segments =
        (state == S1) ? 7'b0110000 : // 1
        (state == S2) ? 7'b1101101 : // 2
        (state == S3) ? 7'b1111001 : // 3
                        7'b0000000;

    wire [6:0] visible_segments =
        allseg_en ? 7'b1111111 :
        (!normal_segment_on || (blink_en && blink_phase)) ? 7'b0000000 :
        digit_segments;

    wire seg_a_area = (pix_x >= 10'd250) && (pix_x < 10'd390) &&
                      (pix_y >= 10'd100) && (pix_y < 10'd116);
    wire seg_g_area = (pix_x >= 10'd250) && (pix_x < 10'd390) &&
                      (pix_y >= 10'd230) && (pix_y < 10'd246);
    wire seg_d_area = (pix_x >= 10'd250) && (pix_x < 10'd390) &&
                      (pix_y >= 10'd360) && (pix_y < 10'd376);

    wire seg_f_area = (pix_x >= 10'd235) && (pix_x < 10'd251) &&
                      (pix_y >= 10'd116) && (pix_y < 10'd230);
    wire seg_b_area = (pix_x >= 10'd389) && (pix_x < 10'd405) &&
                      (pix_y >= 10'd116) && (pix_y < 10'd230);
    wire seg_e_area = (pix_x >= 10'd235) && (pix_x < 10'd251) &&
                      (pix_y >= 10'd246) && (pix_y < 10'd360);
    wire seg_c_area = (pix_x >= 10'd389) && (pix_x < 10'd405) &&
                      (pix_y >= 10'd246) && (pix_y < 10'd360);

    wire segment_pixel =
        (visible_segments[6] && seg_a_area) ||
        (visible_segments[5] && seg_b_area) ||
        (visible_segments[4] && seg_c_area) ||
        (visible_segments[3] && seg_d_area) ||
        (visible_segments[2] && seg_e_area) ||
        (visible_segments[1] && seg_f_area) ||
        (visible_segments[0] && seg_g_area);

    reg [1:0] red;
    reg [1:0] green;
    reg [1:0] blue;

    always @(*) begin
        red   = 2'b00;
        green = 2'b00;
        blue  = 2'b00;

        if (display_on && segment_pixel) begin
            if (allseg_en) begin
                red   = 2'b11;
                green = 2'b11;
                blue  = 2'b11;
            end else begin
                case (state)
                    S1: red   = 2'b11;
                    S2: begin
                        red   = 2'b11;
                        green = 2'b11;
                    end
                    S3: green = 2'b11;
                    default: begin
                        red   = 2'b00;
                        green = 2'b00;
                        blue  = 2'b00;
                    end
                endcase
            end
        end
    end

    // TinyVGA PMOD mapping.
    assign uo_out  = {hsync, blue[0], green[0], red[0],
                      vsync, blue[1], green[1], red[1]};

    assign uio_out = 8'h00;
    assign uio_oe  = 8'h00;

    wire _unused = &{ena, uio_in, 1'b0};

endmodule
