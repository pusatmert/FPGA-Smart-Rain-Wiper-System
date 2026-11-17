# FPGA-Smart-Rain-Wiper-System
├── README.md                  (Projenin vitrini - az önce hazırladığımız metin)
├── src/                       (Kaynak kodlar)
│   ├── top_module.vhd 
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.NUMERIC_STD.ALL;

entity rain_system_top is
    Port (
        clk          : in  STD_LOGIC;                     -- 100 MHz Dahili Basys 3 Saati (W5)
        rst          : in  STD_LOGIC;                     -- Reset Butonu btnC (U18)
        rain_sensor  : in  STD_LOGIC;                     -- Sensör DO Pini (JA2 - L2) [0: Yağmur Var, 1: Yok]
        servo_pwm    : out STD_LOGIC;                     -- SG90 Sinyal Pini (JA1 - J1)
        led_status   : out STD_LOGIC_VECTOR(2 downto 0);  -- Durum LED'leri (LD0, LD1, LD2)
        lcd_rs       : out STD_LOGIC;                     -- LCD RS Pini (JB1 - A14)
        lcd_e        : out STD_LOGIC;                     -- LCD E Pini  (JB2 - A16)
        lcd_data     : out STD_LOGIC_VECTOR(3 downto 0)   -- LCD D4..D7 (JB3, JB4, JB7, JB8)
    );
end rain_system_top;

architecture Behavioral of rain_system_top is

    ----------------------------------------------------------------------------
    -- 1. FSM & YAĞMUR DURUM SİNYALLERİ
    ----------------------------------------------------------------------------
    type system_state_type is (NO_RAIN, RAIN_DETECTED);
    signal current_state, next_state : system_state_type := NO_RAIN;
    signal rain_active : std_logic := '0';

    ----------------------------------------------------------------------------
    -- 2. SERVO PWM SABİTLERİ VE SİNYALLERİ (100 MHz CLK)
    ----------------------------------------------------------------------------
    -- 20 ms Periyot = 2,000,000 clock tick (50 Hz)
    constant PWM_PERIOD   : integer := 2_000_000;
    
    -- SG90 Pals Aralıkları (0.75 ms -> -45°, 1.5 ms -> 0°, 2.25 ms -> +45°)
    constant SERVO_MIN    : integer := 75_000;   -- 0.75 ms
    constant SERVO_CENTER : integer := 150_000;  -- 1.50 ms
    constant SERVO_MAX    : integer := 225_000;  -- 2.25 ms

    signal pwm_counter   : integer range 0 to PWM_PERIOD := 0;
    signal pulse_width   : integer range SERVO_MIN to SERVO_MAX := SERVO_CENTER;
    signal sweep_dir     : std_logic := '1'; -- '1': Sağa doğru artış, '0': Sola doğru azalış
    
    -- Silecek Süpürme Hız Sayacı (~50 Hz güncelleme adımı)
    signal speed_counter : integer range 0 to 300_000 := 0;
    constant SPEED_LIMIT : integer := 150_000; -- Hız ayarı (Düşürülürse silecek hızlanır)

    ----------------------------------------------------------------------------
    -- 3. LCD KONTROL SİNYALLERİ VE METİN SABİTLERİ (4-BIT HD44780)
    ----------------------------------------------------------------------------
    signal clk_div_lcd   : integer range 0 to 100_000 := 0;
    signal tick_1ms      : std_logic := '0';

    type lcd_fsm_type is (LCD_INIT, LCD_WRITE_CMD, LCD_WRITE_DATA, LCD_HOLD);
    signal lcd_state     : lcd_fsm_type := LCD_INIT;

    signal init_step     : integer range 0 to 60 := 0;
    signal char_idx      : integer range 0 to 15 := 0;
    signal nibble_flag   : std_logic := '0';
    signal hold_counter  : integer range 0 to 500 := 0;

    type msg_array is array (0 to 15) of std_logic_vector(7 downto 0);
    
    -- "NO RAIN DETECTED"
    constant MSG_NO_RAIN : msg_array := (
        x"4E", x"4F", x"20", x"52", x"41", x"49", x"4E", x"20",
        x"44", x"45", x"54", x"45", x"43", x"54", x"45", x"44"
    );

    -- "RAIN DETECTED!  "
    constant MSG_RAIN : msg_array := (
        x"52", x"41", x"49", x"4E", x"20", x"44", x"45", x"54",
        x"45", x"43", x"54", x"45", x"44", x"21", x"20", x"20"
    );

    signal current_char  : std_logic_vector(7 downto 0) := x"20";

begin

    ----------------------------------------------------------------------------
    -- [FSM] DURUM MAKİNESİ (Girdi Filtreleme ve Durum Geçişleri)
    ----------------------------------------------------------------------------
    process(clk, rst)
    begin
        if rst = '1' then
            current_state <= NO_RAIN;
        elsif rising_edge(clk) then
            current_state <= next_state;
        end if;
    end process;

    process(current_state, rain_sensor)
    begin
        case current_state is
            when NO_RAIN =>
                if rain_sensor = '0' then -- Aktif Low (Su algılandı)
                    next_state <= RAIN_DETECTED;
                else
                    next_state <= NO_RAIN;
                end if;

            when RAIN_DETECTED =>
                if rain_sensor = '1' then -- Kurulandı
                    next_state <= NO_RAIN;
                else
                    next_state <= RAIN_DETECTED;
                end if;
        end case;
    end process;

    rain_active <= '1' when current_state = RAIN_DETECTED else '0';

    -- Durum LED'leri
    led_status(0) <= '1' when current_state = NO_RAIN else '0';       -- LD0 (Yeşil / Yağmur Yok)
    led_status(1) <= '1' when current_state = RAIN_DETECTED else '0'; -- LD1 (Kırmızı / Yağmur Var)
    led_status(2) <= '0';

    ----------------------------------------------------------------------------
    -- [SERVO] SİLECEK AÇI VE HAREKET MANTIĞI
    ----------------------------------------------------------------------------
    process(clk, rst)
    begin
        if rst = '1' then
            pulse_width   <= SERVO_CENTER;
            speed_counter <= 0;
            sweep_dir     <= '1';
        elsif rising_edge(clk) then
            if current_state = NO_RAIN then
                pulse_width   <= SERVO_CENTER; -- Başlangıç konumu (0 derece)
                speed_counter <= 0;
            else
                -- Yağmur algılandığında periyodik -45° ile +45° arası süpürme
                if speed_counter >= SPEED_LIMIT then
                    speed_counter <= 0;
                    if sweep_dir = '1' then
                        if pulse_width < SERVO_MAX then
                            pulse_width <= pulse_width + 1000;
                        else
                            sweep_dir <= '0';
                        end if;
                    else
                        if pulse_width > SERVO_MIN then
                            pulse_width <= pulse_width - 1000;
                        else
                            sweep_dir <= '1';
                        end if;
                    end if;
                else
                    speed_counter <= speed_counter + 1;
                end if;
            end if;
        end if;
    end process;

    -- 50 Hz PWM Sinyal Üretimi
    process(clk, rst)
    begin
        if rst = '1' then
            pwm_counter <= 0;
            servo_pwm   <= '0';
        elsif rising_edge(clk) then
            if pwm_counter < PWM_PERIOD - 1 then
                pwm_counter <= pwm_counter + 1;
            else
                pwm_counter <= 0;
            end if;

            if pwm_counter < pulse_width then
                servo_pwm <= '1';
            else
                servo_pwm <= '0';
            end if;
        end if;
    end process;

    ----------------------------------------------------------------------------
    -- [LCD] 1 ms ZAMANLAMA VE HD44780 4-BIT KONTROLÜ
    ----------------------------------------------------------------------------
    process(clk)
    begin
        if rising_edge(clk) then
            if clk_div_lcd = 100_000 - 1 then
                clk_div_lcd <= 0;
                tick_1ms    <= '1';
            else
                clk_div_lcd <= clk_div_lcd + 1;
                tick_1ms    <= '0';
            end if;
        end if;
    end process;

    process(clk, rst)
    begin
        if rst = '1' then
            lcd_state    <= LCD_INIT;
            init_step    <= 0;
            char_idx     <= 0;
            nibble_flag  <= '0';
            hold_counter <= 0;
            lcd_rs       <= '0';
            lcd_e        <= '0';
            lcd_data     <= "0000";
        elsif rising_edge(clk) then
            if tick_1ms = '1' then
                case lcd_state is

                    -- Açılış Başlatma Sıralaması (Power-on Reset)
                    when LCD_INIT =>
                        case init_step is
                            when 15 => lcd_data <= "0011"; lcd_e <= '1'; lcd_rs <= '0';
                            when 16 => lcd_e <= '0';
                            when 21 => lcd_data <= "0011"; lcd_e <= '1';
                            when 22 => lcd_e <= '0';
                            when 25 => lcd_data <= "0011"; lcd_e <= '1';
                            when 26 => lcd_e <= '0';
                            when 30 => lcd_data <= "0010"; lcd_e <= '1'; -- 4-bit Modu Aktif Et
                            when 31 => lcd_e <= '0';
                            when 35 =>
                                init_step <= 0;
                                lcd_state <= LCD_WRITE_DATA;
                            when others => null;
                        end case;
                        if lcd_state = LCD_INIT then
                            init_step <= init_step + 1;
                        end if;

                    -- Ekrana Karakter Basma Döngüsü
                    when LCD_WRITE_DATA =>
                        if rain_active = '0' then
                            current_char <= MSG_NO_RAIN(char_idx);
                        else
                            current_char <= MSG_RAIN(char_idx);
                        end if;

                        if nibble_flag = '0' then
                            lcd_rs      <= '1';
                            lcd_data    <= current_char(7 downto 4); -- Üst 4 bit
                            lcd_e       <= '1';
                            nibble_flag <= '1';
                        else
                            lcd_data    <= current_char(3 downto 0); -- Alt 4 bit
                            lcd_e       <= '0';
                            nibble_flag <= '0';
                            if char_idx = 15 then
                                char_idx  <= 0;
                                lcd_state <= LCD_HOLD;
                            else
                                char_idx  <= char_idx + 1;
                            end if;
                        end if;

                    -- Yazı Yenileme Bekleme Durumu (~250 ms)
                    when LCD_HOLD =>
                        lcd_e <= '0';
                        if hold_counter >= 250 then
                            hold_counter <= 0;
                            lcd_state    <= LCD_WRITE_DATA;
                        else
                            hold_counter <= hold_counter + 1;
                        end if;

                    when others =>
                        lcd_state <= LCD_INIT;
                end case;
            end if;
        end if;
    end process;

end Behavioral;
│   └── constraints.xdc
## 100 MHz Dahili Sistem Saati
set_property PACKAGE_PIN W5 [get_ports clk]							
	set_property IOSTANDARD LVCMOS33 [get_ports clk]
	create_clock -add -name sys_clk_pin -period 10.00 -waveform {0 5} [get_ports clk]

## Reset Butonu (Orta Buton - btnC)
set_property PACKAGE_PIN U18 [get_ports rst]						
	set_property IOSTANDARD LVCMOS33 [get_ports rst]

## Durum LED'leri (LD0, LD1, LD2)
set_property PACKAGE_PIN U16 [get_ports {led_status[0]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {led_status[0]}]
set_property PACKAGE_PIN E19 [get_ports {led_status[1]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {led_status[1]}]
set_property PACKAGE_PIN U19 [get_ports {led_status[2]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {led_status[2]}]

## PMOD Header JA (Sensör ve Servo)
## JA1 (Pin 1) -> SG90 Servo Sinyal Pini
set_property PACKAGE_PIN J1 [get_ports servo_pwm]					
	set_property IOSTANDARD LVCMOS33 [get_ports servo_pwm]

## JA2 (Pin 2) -> Yağmur Sensörü DO (Dijital Çıkış)
set_property PACKAGE_PIN L2 [get_ports rain_sensor]					
	set_property IOSTANDARD LVCMOS33 [get_ports rain_sensor]

## PMOD Header JB (16x1 HD44780 LCD Sinyalleri)
## JB1 -> LCD RS (Pin 4)
set_property PACKAGE_PIN A14 [get_ports lcd_rs]					
	set_property IOSTANDARD LVCMOS33 [get_ports lcd_rs]

## JB2 -> LCD E (Pin 6)
set_property PACKAGE_PIN A16 [get_ports lcd_e]					
	set_property IOSTANDARD LVCMOS33 [get_ports lcd_e]

## JB3 -> LCD D4 (Pin 11)
set_property PACKAGE_PIN B15 [get_ports {lcd_data[0]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data[0]}]

## JB4 -> LCD D5 (Pin 12)
set_property PACKAGE_PIN B16 [get_ports {lcd_data[1]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data[1]}]

## JB7 -> LCD D6 (Pin 13)
set_property PACKAGE_PIN A15 [get_ports {lcd_data[2]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data[2]}]

## JB8 -> LCD D7 (Pin 14)
set_property PACKAGE_PIN A17 [get_ports {lcd_data[3]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data[3]}]
└── images/                    (Görseller)
    ├── vivado_sim.png         (Vivado simülasyon / blok diyagram görüntüsü)
    └── prototype_setup.jpg    (Basys 3, sensör ve LCD bağlantı fotoğrafı)
