SE520 Laboratory

**SE520 - Cryptography and secure protocols for embedded systems**

# Hardware Implementation of Finite-Field Arithmetic

*Lucas Nidiau*
*Alexandre Tisserand*


15/01/2021

## 1 mod m adder

### 1.1

Below is VHDL behavioral description of the modular addition. 
It's not syntheisable because of the usage of mod operator.

``` vhdl
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.numeric_std.ALL;
use STD.textio.ALL;

entity modm_addition is
generic(k:integer:=8);
port(	x, y, m 	: in std_logic_vector(k-1 downto 0);
		z		: out std_logic_vector(k-1 downto 0));
end modm_addition;

architecture Behavioral of modm_addition is

begin

process(x,y)
variable z1, z2 : integer;
variable c1, c2 : integer;
variable L : line;
begin

z1:=to_integer(unsigned('0'&x)+unsigned(y));
z2:=(z1 mod 2**k) + (2**k - to_integer(unsigned(m)));

c1:= z1/2**k;
c2:= z2/2**k; 

if c1 = 0 and c2 = 0 then
	z <= std_logic_vector(to_unsigned(z1 mod 2**k,k));
else
	z <= std_logic_vector(to_unsigned(z2 mod 2**k,k));
end if;
write(L,"z1 =");
write(L,z1);
write(L," z2 =");
write(L,z2);
write(L," c1 =");
write(L,c1);
write(L," c2 =");
write(L,c2);
writeline(OUTPUT,L);
end process;
end Behavioral;
```

We use the following test bench to verify that the operation is corectelty executed:

```vhdl
LIBRARY ieee;
USE ieee.std_logic_1164.ALL;
use IEEE.numeric_std.ALL;

ENTITY tb_modm_addition IS
END tb_modm_addition;
 
ARCHITECTURE behavior OF tb_modm_addition IS 
 
    -- Component Declaration for the Unit Under Test (UUT)
 
    COMPONENT modm_adder_to_be_completed
    PORT(
         x : IN  std_logic_vector(7 downto 0);
         y : IN  std_logic_vector(7 downto 0);
         --m : IN  std_logic_vector(7 downto 0);
         z : OUT  std_logic_vector(7 downto 0)
        );
    END COMPONENT;
    

   --Inputs
   signal x : std_logic_vector(7 downto 0) := (others => '0');
   signal y : std_logic_vector(7 downto 0) := (others => '0');
  -- signal m : std_logic_vector(7 downto 0) := (others => '0');
   signal z : std_logic_vector(7 downto 0) := (others => '0');
   
 
BEGIN
 
	-- Instantiate the Unit Under Test (UUT)
   uut: modm_adder_to_be_completed PORT MAP (
          x => x,
          y => y,
        --  m => m,
          z => z
        );


   -- Stimulus process
   stim_proc: process
   begin		
		x <= std_logic_vector(to_unsigned(129, 8));
		y <= std_logic_vector(to_unsigned(105, 8));		
	--	m <= std_logic_vector(to_unsigned(239, 8));
      wait for 100 ns;	
		x <= std_logic_vector(to_unsigned(234, 8));
		y <= std_logic_vector(to_unsigned(238, 8));
      wait for 100 ns;	
		x <= std_logic_vector(to_unsigned(215, 8));
		y <= std_logic_vector(to_unsigned(35, 8));

      wait;
   end process;

END;
```
Here is the ouput of the simulation : 

Comme le montre cette simulation, ce code produit bien le resulat attendu pour les opération:

- 129 + 105 mod 239
- 234 + 238 mod 239

En revenche nous ne somme pas parvenu a effectuer l'opération 215 + 35 mod 239 a partir de la trame de modm_addition_to_be_completed.vhd . 

Afin d'obtenir un module d'addition fonctionnel nous avons repris l'exemple adder_subtractor.vhd fourni par le site http://www.arithmetic-circuits.org/ ainsi que son test bench test_add_sub.vhd que nous avons modifier afin de répondre a nos attente.

Avec les codes suivants : 

```vhdl
----------------------------------------------------------------------------
-- adder subtractor mod M (adder_subtractor.vhd)
--
-- Adds or Subtract two K-bits operands X and Y mod M
----------------------------------------------------------------------------
library IEEE;
use IEEE.std_logic_1164.all;
use IEEE.std_logic_arith.all;
use IEEE.std_logic_unsigned.all;
package my_package is
  constant K: integer := 8;
  constant M: std_logic_vector(K-1 downto 0) := conv_std_logic_vector(239, K);
end my_package;

library ieee; 
use ieee.std_logic_1164.all;
use ieee.std_logic_arith.all;
use ieee.std_logic_unsigned.all;
use work.my_package.all;

entity adder_subtractor is
port (
  x, y: in std_logic_vector(K-1 downto 0);
  addb_sub: in std_logic;
  z: out std_logic_vector(K-1 downto 0)
);
end adder_subtractor;

architecture rtl of adder_subtractor is
  signal long_x, xor_y, sum1, long_z1, xor_m, sum2: std_logic_vector(K downto 0);
  signal c1, c2, sel: std_logic;
  signal z1, z2: std_logic_vector(K-1 downto 0);

begin

  long_x <= '0' & x;
  xor_gates1: for i in 0 to K-1 generate 
      xor_y(i) <= y(i) xor addb_sub; 
  end generate;
  xor_y(K) <= '0';
  sum1 <= addb_sub + long_x + xor_y;
  c1 <= sum1(K);
  z1 <= sum1(K-1 downto 0);
  long_z1 <= '0' & z1;
  xor_gates2: for i in 0 to k-1 generate
      xor_m(i) <= m(i) xor not(addb_sub);
  end generate;
  xor_m(k) <= '0';
  sum2 <= not(addb_sub) + long_z1 + xor_m;
  c2 <= sum2(k);
  z2 <= sum2(k-1 downto 0);
  sel <= (not(addb_sub) and (c1 or c2)) or (addb_sub and not(c1));
  with sel select z <= z1 when '0', z2 when others;

end rtl;

```
Et le test bench suivant :

```vhdl
--------------------------------------------------------------------------------
-- 
-- VHDL Test Bench for module: adder_subtractor.vhd
--
-- Executes an exhaustive Test Bench for mod M adder substractor
--
--------------------------------------------------------------------------------
LIBRARY ieee;
USE ieee.std_logic_1164.ALL;
USE IEEE.std_logic_arith.all;
USE ieee.std_logic_signed.all;
USE ieee.std_logic_unsigned.all;
USE ieee.numeric_std.ALL;
USE ieee.std_logic_textio.ALL;
USE std.textio.ALL;
use work.my_package.all;


ENTITY test_add_sub IS
END test_add_sub;

ARCHITECTURE behavior OF test_add_sub IS 

  -- Component Declaration for the Unit Under Test (UUT)
  COMPONENT adder_subtractor
  PORT(
    x,y : in std_logic_vector(K-1 downto 0);
    addb_sub: in std_logic;
    z: out std_logic_vector(K-1 downto 0)
    );
  END COMPONENT;

  --Inputs
  SIGNAL x, y :  std_logic_vector(K-1 downto 0) := (others=>'0');
  SIGNAL addb_sub :  std_logic;
  --Outputs
  SIGNAL z :  std_logic_vector(K-1 downto 0);

  constant DELAY : time := 100 ns;

BEGIN

  -- Instantiate the Unit Under Test (UUT)
  uut: adder_subtractor PORT MAP(
    x => x,
    y => y,
    addb_sub => addb_sub,
    z => z
  );
  
  stim_proc: process
   begin	
        addb_sub <= '0';
		x <= std_logic_vector(to_unsigned(129, 8));
		y <= std_logic_vector(to_unsigned(105, 8));		
      wait for 100 ns;	
		x <= std_logic_vector(to_unsigned(234, 8));
		y <= std_logic_vector(to_unsigned(238, 8));
      wait for 100 ns;	
		x <= std_logic_vector(to_unsigned(215, 8));
		y <= std_logic_vector(to_unsigned(35, 8));

      wait;
   end process;
END;
```
On obtient

![Q1ok](C:\Users\SESA458137\OneDrive - Schneider Electric\Travail\ESISAR\S6\SE520 Crypto\vhdl\Q1ok.JPG)

Ce code nous donne bien le resultat attendu pour toutes les opréation demander en plus d'etre synthéisable.

### 1.2

L'utilisation de carry adders permet de crée la structure d'addition la plus simple possible. En revenche c'est egalement la structure la plus lente en raison de la propagation de la retenue dans l'ensemble des full adder.
De ce fait plus le nombre de bits est important et plus le temps de propagation est long. 

De ce fait et avec 2 structure de Carry Ripple on a le temps de propagation de :
$$
Computation\ time=(k+1)T_{FA} + T_{MUX2}
$$

### 1.3

Voici une implementation du datapath

```vhdl
library IEEE;
use IEEE.std_logic_1164.all;
use IEEE.std_logic_arith.all;
use IEEE.std_logic_unsigned.all;


package dar_modm_multiplier_parameters is
  constant K: integer := 8;
  constant logK: integer := 3;
  constant integer_M: integer := 239;
  constant M: std_logic_vector(K-1 downto 0) := conv_std_logic_vector(integer_M, K);
  constant minus_M: std_logic_vector(K-1 downto 0) := conv_std_logic_vector(2**K - integer_M, K);
  constant ZERO: std_logic_vector(logK-1 downto 0) := (others => '0');
end dar_modm_multiplier_parameters;


library ieee; 
use ieee.std_logic_1164.all;
use ieee.std_logic_arith.all;
use ieee.std_logic_unsigned.all;
use ieee.numeric_std.all;
use work.dar_modm_multiplier_parameters.all;

entity modm_adder_to_be_completed is
port (
  x, y: in std_logic_vector(K-1 downto 0);
  z: out std_logic_vector(K-1 downto 0)
);
end modm_adder_to_be_completed;

architecture rtl of modm_adder_to_be_completed is
  signal long_x, sum1, long_z1, sum2, debug: std_logic_vector(K downto 0);
  signal c1, c2, sel: std_logic;
  signal pow: std_logic_vector(K downto 0) ; 
  signal z1, z2: std_logic_vector(K-1 downto 0);

begin

--

  long_x <= '0' & x;
  sum1 <= long_x + y;
  c1 <= sum1(K);
  z1 <= sum1(K-1 downto 0);
-- to be completed
  long_z1 <= '0' & z1;
  sum2 <= long_z1 + minus_M;
  c2 <= sum2(K);
  z2 <= sum2(K-1 downto 0);


  --  pow <=  (K => '1', others => '0');
 --   long_x <= '0' & x;
  --  sum1 <= long_x + y;
  --  c1 <= sum1(K);
 --   z1 <= sum1(K-1 downto 0);
    
  --  long_z1 <= '0' & z1;
   -- sum2 <= std_logic_vector(unsigned(z1)) mod(2**K-integer_M);
   
    -- sum2 <= ( to_unsigned(2**K,K-1) - to_unsigned(integer_M,K-1) );
     
   -- debug <= pow)) - to_integer(unsigned('0'&M))),K-1)  );
   
----sum2 <= long_z1 + (pow - '0'&M);
  --  c2 <= sum2(K);
  --  z2 <= sum2(K-1 downto 0);
    
    sel <= c1 or c2;
    with sel select z <= z1 when '0', z2 when others;

end rtl;
```

### 1.4

Here is the completed test bench to ensure the behaving of previous adder. 

```vhdl
LIBRARY ieee;
USE ieee.std_logic_1164.ALL;
use IEEE.numeric_std.ALL;

ENTITY tb_modm_adder IS
END tb_modm_adder;
 
ARCHITECTURE Behaviorial OF tb_modm_adder IS 
 
    -- Component Declaration for the Unit Under Test (UUT)
 
    COMPONENT modm_adder_to_be_completed
    PORT(
         x : IN  std_logic_vector(7 downto 0);
         y : IN  std_logic_vector(7 downto 0);
         --m : IN  std_logic_vector(7 downto 0);
         z : OUT  std_logic_vector(7 downto 0)
        -- clk, reset, start: in std_logic;
        -- done: out std_logic
        );
    END COMPONENT;
    


   --Inputs
   signal x : std_logic_vector(7 downto 0) := (others => '0');
   signal y : std_logic_vector(7 downto 0) := (others => '0');
  -- signal m : std_logic_vector(7 downto 0) := (others => '0');
   signal z : std_logic_vector(7 downto 0) := (others => '0');

 
BEGIN
 
	-- Instantiate the Unit Under Test (UUT)
   uut: modm_adder_to_be_completed
   PORT MAP (
          x => x,
          y => y,

        --  m => m,
          z => z
        );


   -- Stimulus process
   stim_proc: process
   begin	
		x <= std_logic_vector(to_unsigned(129, 8));
		y <= std_logic_vector(to_unsigned(105, 8));		
	--	m <= std_logic_vector(to_unsigned(239, 8));
      wait for 100 ns;	
		x <= std_logic_vector(to_unsigned(234, 8));
		y <= std_logic_vector(to_unsigned(238, 8));
      wait for 100 ns;	
		x <= std_logic_vector(to_unsigned(215, 8));
		y <= std_logic_vector(to_unsigned(35, 8));

      wait;
   end process;

END;

```
Correct modular addition is confirmed.

![Q14](C:\Users\SESA458137\OneDrive - Schneider Electric\Travail\ESISAR\S6\SE520 Crypto\vhdl\Q14.JPG)

### 1.5

