# HDF5 File Structure: /home/hyoklee/neuroh5/data/dentate_test.h5


## Attributes

- **ahdf5-smd-H5Types:** `BEST GUESS: Type registry group containing committed HDF5 compound and enumerated datatypes shared across all populations and projections in this dentate gyrus circuit file.` (type: `str`)
- **ahdf5-smd-Projections:** `BEST GUESS: Top-level group containing all synaptic projections organised by source population. Each first-level child names a source cell type; each second-level child names a destination cell type.` (type: `str`)
- **ahdf5-smd-root:** `BEST GUESS: neuroh5 HDF5 connectivity file representing a dentate gyrus microcircuit with 11 neuron populations across 6 cell types (GC, MC, AAC, BC, HC, NGFC) and 40 valid synaptic projections stored in Destination Block Sparse format.` (type: `str`)


## Group: /H5Types

### Attributes

- **ahdf5-smd-Population labels:** `BEST GUESS: Committed enumerated HDF5 datatype (uint16) mapping integer codes to dentate gyrus cell-type names: Granule Cell (GC), Mossy Cell (MC), Axo-Axonic Cell (AAC), Basket Cell (BC), HIPP Cell (HC), Neurogliaform Cell (NGFC).` (type: `str`)
- **ahdf5-smd-Population projections:** `BEST GUESS: Committed compound HDF5 datatype with fields Source (uint16) and Destination (uint16) describing a directed synaptic projection between two neuron populations.` (type: `str`)
- **ahdf5-smd-Population range:** `BEST GUESS: Committed compound HDF5 datatype with fields Start (uint64), Count (uint32), and Population (uint16) defining the global ID (GID) range for one neuron population.` (type: `str`)
- **ahdf5-smd-Populations:** `BEST GUESS: Registry dataset (shape 11) listing all 11 neuron populations in the dentate gyrus circuit. Each record gives the starting GID, neuron count, and population label.` (type: `str`)
- **ahdf5-smd-Valid population projections:** `BEST GUESS: Registry dataset (shape 40) enumerating all 40 valid directed (Source, Destination) population projection pairs in this dentate gyrus circuit.` (type: `str`)


### Dataset: Populations

#### Properties

- **Shape:** `(11,)`
- **Data Type:** `[('Start', '<u8'), ('Count', '<u4'), ('Population', '<u2')]`
- **Size:** `11` elements
- **Chunks:** `(512,)`

**Data (Key-Value Format):**

- `index_0`: `(0, 1000000, 0)`
- `index_1`: `(1000000, 30000, 1)`
- `index_2`: `(1030000, 9000, 2)`
- `index_3`: `(1039000, 3800, 3)`
- `index_4`: `(1042800, 450, 4)`
- `index_5`: `(1043250, 1400, 5)`
- `index_6`: `(1044650, 5000, 6)`
- `index_7`: `(1049650, 3000, 7)`
- `index_8`: `(1052650, 3000, 8)`
- `index_9`: `(1093650, 34000, 10)`
- *(showing 10 of 11 rows using 'uniform' sampling)*


### Dataset: Valid population projections

#### Properties

- **Shape:** `(40,)`
- **Data Type:** `[('Source', '<u2'), ('Destination', '<u2')]`
- **Size:** `40` elements
- **Chunks:** `(1024,)`

**Data (Key-Value Format):**

- `index_0`: `(10, 6)`
- `index_1`: `(10, 0)`
- `index_2`: `(3, 0)`
- `index_3`: `(0, 1)`
- `index_4`: `(4, 1)`
- `index_5`: `(2, 2)`
- `index_6`: `(3, 3)`
- `index_7`: `(1, 4)`
- `index_8`: `(1, 5)`
- `index_9`: `(6, 6)`
- *(showing 10 of 40 rows using 'uniform' sampling)*


## Group: /Projections

### Attributes

- **ahdf5-smd-AAC:** `BEST GUESS: Projections originating from Axo-Axonic Cell (AAC) interneurons. AACs target the axon initial segment of principal and other neurons, providing powerful inhibitory control of spike initiation.` (type: `str`)
- **ahdf5-smd-BC:** `BEST GUESS: Projections originating from Basket Cell (BC) interneurons. Basket cells provide perisomatic inhibition with fast, precise timing control over principal neurons.` (type: `str`)
- **ahdf5-smd-GC:** `BEST GUESS: Projections originating from Granule Cell (GC) principal neurons. GCs are the primary excitatory output of the dentate gyrus, targeting multiple interneuron types via mossy fibre collaterals.` (type: `str`)
- **ahdf5-smd-HC:** `BEST GUESS: Projections originating from HIPP Cell (HC) inhibitory interneurons. HIPPs are dendrite-targeting interneurons that modulate distal dendritic integration in granule cells.` (type: `str`)
- **ahdf5-smd-MC:** `BEST GUESS: Projections originating from Mossy Cell (MC) excitatory interneurons. Mossy cells provide recurrent excitation to granule cells and drive feedforward inhibition via interneurons.` (type: `str`)
- **ahdf5-smd-NGFC:** `BEST GUESS: Projections originating from Neurogliaform Cell (NGFC) interneurons. NGFCs mediate slow, diffuse GABAergic inhibition through volume transmission, modulating large dendritic territories.` (type: `str`)


### Group: /Projections/AAC

#### Attributes

- **ahdf5-smd-GC:** `BEST GUESS: AAC→GC projection: Axo-Axonic Cell synapses onto Granule Cell axon initial segments. Contains Edges (DBS connectivity) and Attributes (distance) subgroups.` (type: `str`)
- **ahdf5-smd-MC:** `BEST GUESS: AAC→MC projection: Axo-Axonic Cell synapses onto Mossy Cell axon initial segments.` (type: `str`)
- **ahdf5-smd-NGFC:** `BEST GUESS: AAC→NGFC projection: Axo-Axonic Cell synapses onto Neurogliaform Cell targets.` (type: `str`)


#### Group: /Projections/AAC/GC

##### Attributes

- **ahdf5-smd-Attributes:** `BEST GUESS: Per-edge spatial distance attributes for the AAC→GC projection.` (type: `str`)
- **ahdf5-smd-Edges:** `BEST GUESS: Destination Block Sparse connectivity arrays for the AAC→GC projection.` (type: `str`)


##### Group: /Projections/AAC/GC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(15,)`
- **Data Type:** `float32`
- **Size:** `15` elements

**Data (Key-Value Format):**

- `index_0`: `125.44959259033203`
- `index_1`: `172.80284118652344`
- `index_2`: `362.97637939453125`
- `index_3`: `78.737060546875`
- `index_4`: `324.82647705078125`
- `index_5`: `89.07804107666016`
- `index_6`: `50.36003494262695`
- `index_7`: `79.30327606201172`
- `index_8`: `322.1920471191406`
- `index_9`: `16.107223510742188`
- *(showing 10 of 15 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(15,)`
- **Data Type:** `float32`
- **Size:** `15` elements

**Data (Key-Value Format):**

- `index_0`: `99.70726013183594`
- `index_1`: `90.21533203125`
- `index_2`: `164.84410095214844`
- `index_3`: `90.11481475830078`
- `index_4`: `115.64576721191406`
- `index_5`: `110.77388000488281`
- `index_6`: `237.11602783203125`
- `index_7`: `215.83074951171875`
- `index_8`: `108.96539306640625`
- `index_9`: `94.79337310791016`
- *(showing 10 of 15 rows using 'uniform' sampling)*


##### Group: /Projections/AAC/GC/Edges

###### Attributes

- **ahdf5-smd-Destination Block Index:** `BEST GUESS: First destination GID of each contiguous destination block in the AAC→GC DBS representation.` (type: `str`)
- **ahdf5-smd-Destination Block Pointer:** `BEST GUESS: Offsets into the Destination Pointer array marking the start of each destination block for the AAC→GC projection.` (type: `str`)
- **ahdf5-smd-Destination Pointer:** `BEST GUESS: Offsets into Source Index for each GC destination in the AAC→GC projection.` (type: `str`)
- **ahdf5-smd-Source Index:** `BEST GUESS: Flat array of source AAC GIDs for all edges in the AAC→GC projection (uint32). Length equals the total number of synapses.` (type: `str`)


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(8,)`
- **Data Type:** `uint32`
- **Size:** `8` elements
- **Chunks:** `(8,)`

**Data (Key-Value Format):**

- `index_0`: `69`
- `index_1`: `72`
- `index_2`: `79`
- `index_3`: `82`
- `index_4`: `266`
- `index_5`: `269`
- `index_6`: `277`
- `index_7`: `285`


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(9,)`
- **Data Type:** `uint64`
- **Size:** `9` elements
- **Chunks:** `(9,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `1`
- `index_2`: `2`
- `index_3`: `3`
- `index_4`: `5`
- `index_5`: `6`
- `index_6`: `7`
- `index_7`: `13`
- `index_8`: `15`


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(15,)`
- **Data Type:** `uint64`
- **Size:** `15` elements
- **Chunks:** `(15,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `1`
- `index_2`: `3`
- `index_3`: `4`
- `index_4`: `6`
- `index_5`: `8`
- `index_6`: `10`
- `index_7`: `11`
- `index_8`: `13`
- `index_9`: `15`
- *(showing 10 of 15 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(15,)`
- **Data Type:** `uint32`
- **Size:** `15` elements
- **Chunks:** `(15,)`

**Data (Key-Value Format):**

- `index_0`: `500000`
- `index_1`: `500006`
- `index_2`: `500007`
- `index_3`: `500002`
- `index_4`: `500004`
- `index_5`: `500008`
- `index_6`: `500005`
- `index_7`: `500007`
- `index_8`: `500009`
- `index_9`: `500001`
- *(showing 10 of 15 rows using 'uniform' sampling)*


#### Group: /Projections/AAC/MC


##### Group: /Projections/AAC/MC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(3975,)`
- **Data Type:** `float32`
- **Size:** `3975` elements

**Data (Key-Value Format):**

- `index_0`: `159.27268981933594`
- `index_1`: `1286.063232421875`
- `index_2`: `491.31610107421875`
- `index_3`: `1533.15625`
- `index_4`: `1022.8267822265625`
- `index_5`: `1720.11279296875`
- `index_6`: `393.5653076171875`
- `index_7`: `1122.0023193359375`
- `index_8`: `1919.9248046875`
- `index_9`: `41.6798210144043`
- *(showing 10 of 3975 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(3975,)`
- **Data Type:** `float32`
- **Size:** `3975` elements

**Data (Key-Value Format):**

- `index_0`: `1.0`
- `index_1`: `1.0`
- `index_2`: `1.0`
- `index_3`: `1.0`
- `index_4`: `1.0`
- `index_5`: `1.0`
- `index_6`: `1.0`
- `index_7`: `1.0`
- `index_8`: `1.0`
- `index_9`: `1.0`
- *(showing 10 of 3975 rows using 'uniform' sampling)*


##### Group: /Projections/AAC/MC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(15,)`
- **Data Type:** `uint32`
- **Size:** `15` elements
- **Chunks:** `(15,)`

**Data (Key-Value Format):**

- `index_0`: `54`
- `index_1`: `58`
- `index_2`: `69`
- `index_3`: `72`
- `index_4`: `78`
- `index_5`: `95`
- `index_6`: `268`
- `index_7`: `275`
- `index_8`: `285`
- `index_9`: `418`
- *(showing 10 of 15 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(16,)`
- **Data Type:** `uint64`
- **Size:** `16` elements
- **Chunks:** `(16,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `1`
- `index_2`: `4`
- `index_3`: `6`
- `index_4`: `7`
- `index_5`: `14`
- `index_6`: `18`
- `index_7`: `19`
- `index_8`: `26`
- `index_9`: `29`
- *(showing 10 of 16 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(29,)`
- **Data Type:** `uint64`
- **Size:** `29` elements
- **Chunks:** `(29,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `429`
- `index_2`: `744`
- `index_3`: `1195`
- `index_4`: `1629`
- `index_5`: `2146`
- `index_6`: `2584`
- `index_7`: `2992`
- `index_8`: `3375`
- `index_9`: `3975`
- *(showing 10 of 29 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(3975,)`
- **Data Type:** `uint32`
- **Size:** `3975` elements
- **Chunks:** `(3975,)`

**Data (Key-Value Format):**

- `index_0`: `820`
- `index_1`: `3040`
- `index_2`: `2043`
- `index_3`: `11034`
- `index_4`: `10668`
- `index_5`: `6434`
- `index_6`: `5871`
- `index_7`: `8061`
- `index_8`: `9660`
- `index_9`: `16687`
- *(showing 10 of 3975 rows using 'uniform' sampling)*


#### Group: /Projections/AAC/NGFC


##### Group: /Projections/AAC/NGFC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(116,)`
- **Data Type:** `float32`
- **Size:** `116` elements

**Data (Key-Value Format):**

- `index_0`: `380.5306091308594`
- `index_1`: `77.5169448852539`
- `index_2`: `286.4912109375`
- `index_3`: `136.23069763183594`
- `index_4`: `380.5306091308594`
- `index_5`: `136.23069763183594`
- `index_6`: `73.78326416015625`
- `index_7`: `136.23069763183594`
- `index_8`: `134.20265197753906`
- `index_9`: `245.355224609375`
- *(showing 10 of 116 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(116,)`
- **Data Type:** `float32`
- **Size:** `116` elements

**Data (Key-Value Format):**

- `index_0`: `258.836669921875`
- `index_1`: `50.18582534790039`
- `index_2`: `209.57998657226562`
- `index_3`: `80.05650329589844`
- `index_4`: `258.836669921875`
- `index_5`: `80.05650329589844`
- `index_6`: `72.07319641113281`
- `index_7`: `80.05650329589844`
- `index_8`: `407.4867248535156`
- `index_9`: `202.22413635253906`
- *(showing 10 of 116 rows using 'uniform' sampling)*


##### Group: /Projections/AAC/NGFC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(14,)`
- **Data Type:** `uint32`
- **Size:** `14` elements
- **Chunks:** `(14,)`

**Data (Key-Value Format):**

- `index_0`: `58`
- `index_1`: `66`
- `index_2`: `69`
- `index_3`: `76`
- `index_4`: `78`
- `index_5`: `265`
- `index_6`: `269`
- `index_7`: `277`
- `index_8`: `282`
- `index_9`: `418`
- *(showing 10 of 14 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(15,)`
- **Data Type:** `uint64`
- **Size:** `15` elements
- **Chunks:** `(15,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `1`
- `index_2`: `4`
- `index_3`: `5`
- `index_4`: `12`
- `index_5`: `13`
- `index_6`: `16`
- `index_7`: `17`
- `index_8`: `22`
- `index_9`: `25`
- *(showing 10 of 15 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(25,)`
- **Data Type:** `uint64`
- **Size:** `25` elements
- **Chunks:** `(25,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `5`
- `index_2`: `14`
- `index_3`: `26`
- `index_4`: `42`
- `index_5`: `53`
- `index_6`: `69`
- `index_7`: `82`
- `index_8`: `94`
- `index_9`: `116`
- *(showing 10 of 25 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(116,)`
- **Data Type:** `uint32`
- **Size:** `116` elements
- **Chunks:** `(116,)`

**Data (Key-Value Format):**

- `index_0`: `1997`
- `index_1`: `1956`
- `index_2`: `2057`
- `index_3`: `1806`
- `index_4`: `2039`
- `index_5`: `2060`
- `index_6`: `1929`
- `index_7`: `1947`
- `index_8`: `1712`
- `index_9`: `2030`
- *(showing 10 of 116 rows using 'uniform' sampling)*


### Group: /Projections/BC


#### Group: /Projections/BC/BC


##### Group: /Projections/BC/BC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(1022,)`
- **Data Type:** `float32`
- **Size:** `1022` elements

**Data (Key-Value Format):**

- `index_0`: `776.1986694335938`
- `index_1`: `462.2174072265625`
- `index_2`: `62.7889518737793`
- `index_3`: `570.0242919921875`
- `index_4`: `127.2589340209961`
- `index_5`: `615.039306640625`
- `index_6`: `935.8656005859375`
- `index_7`: `13.40651798248291`
- `index_8`: `601.5844116210938`
- `index_9`: `730.5845336914062`
- *(showing 10 of 1022 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(1022,)`
- **Data Type:** `float32`
- **Size:** `1022` elements

**Data (Key-Value Format):**

- `index_0`: `198.53445434570312`
- `index_1`: `100.6503677368164`
- `index_2`: `228.59861755371094`
- `index_3`: `101.2039794921875`
- `index_4`: `275.52520751953125`
- `index_5`: `122.86229705810547`
- `index_6`: `182.7291717529297`
- `index_7`: `109.89151000976562`
- `index_8`: `241.48475646972656`
- `index_9`: `434.94012451171875`
- *(showing 10 of 1022 rows using 'uniform' sampling)*


##### Group: /Projections/BC/BC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(55,)`
- **Data Type:** `uint32`
- **Size:** `55` elements
- **Chunks:** `(55,)`

**Data (Key-Value Format):**

- `index_0`: `531`
- `index_1`: `595`
- `index_2`: `644`
- `index_3`: `678`
- `index_4`: `803`
- `index_5`: `2263`
- `index_6`: `2305`
- `index_7`: `2333`
- `index_8`: `2375`
- `index_9`: `3536`
- *(showing 10 of 55 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(56,)`
- **Data Type:** `uint64`
- **Size:** `56` elements
- **Chunks:** `(56,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `7`
- `index_2`: `14`
- `index_3`: `23`
- `index_4`: `30`
- `index_5`: `36`
- `index_6`: `44`
- `index_7`: `50`
- `index_8`: `57`
- `index_9`: `66`
- *(showing 10 of 56 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(66,)`
- **Data Type:** `uint64`
- **Size:** `66` elements
- **Chunks:** `(66,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `75`
- `index_2`: `197`
- `index_3`: `309`
- `index_4`: `441`
- `index_5`: `504`
- `index_6`: `630`
- `index_7`: `749`
- `index_8`: `854`
- `index_9`: `1022`
- *(showing 10 of 66 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(1022,)`
- **Data Type:** `uint32`
- **Size:** `1022` elements
- **Chunks:** `(1022,)`

**Data (Key-Value Format):**

- `index_0`: `595`
- `index_1`: `2355`
- `index_2`: `2290`
- `index_3`: `568`
- `index_4`: `626`
- `index_5`: `678`
- `index_6`: `595`
- `index_7`: `609`
- `index_8`: `699`
- `index_9`: `2380`
- *(showing 10 of 1022 rows using 'uniform' sampling)*


#### Group: /Projections/BC/GC


##### Group: /Projections/BC/GC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(43,)`
- **Data Type:** `float32`
- **Size:** `43` elements

**Data (Key-Value Format):**

- `index_0`: `55.57406997680664`
- `index_1`: `1.9403769969940186`
- `index_2`: `91.60331726074219`
- `index_3`: `90.18262481689453`
- `index_4`: `125.50450897216797`
- `index_5`: `218.3138427734375`
- `index_6`: `131.94366455078125`
- `index_7`: `53.18739700317383`
- `index_8`: `121.24933624267578`
- `index_9`: `61.596561431884766`
- *(showing 10 of 43 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(43,)`
- **Data Type:** `float32`
- **Size:** `43` elements

**Data (Key-Value Format):**

- `index_0`: `132.68753051757812`
- `index_1`: `189.9861602783203`
- `index_2`: `210.65203857421875`
- `index_3`: `157.35768127441406`
- `index_4`: `136.52346801757812`
- `index_5`: `146.3351287841797`
- `index_6`: `142.6179962158203`
- `index_7`: `132.5930633544922`
- `index_8`: `156.55284118652344`
- `index_9`: `225.77052307128906`
- *(showing 10 of 43 rows using 'uniform' sampling)*


##### Group: /Projections/BC/GC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(28,)`
- **Data Type:** `uint32`
- **Size:** `28` elements
- **Chunks:** `(28,)`

**Data (Key-Value Format):**

- `index_0`: `571`
- `index_1`: `609`
- `index_2`: `645`
- `index_3`: `676`
- `index_4`: `694`
- `index_5`: `2263`
- `index_6`: `2305`
- `index_7`: `2327`
- `index_8`: `2339`
- `index_9`: `2389`
- *(showing 10 of 28 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(29,)`
- **Data Type:** `uint64`
- **Size:** `29` elements
- **Chunks:** `(29,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `3`
- `index_2`: `6`
- `index_3`: `10`
- `index_4`: `13`
- `index_5`: `16`
- `index_6`: `19`
- `index_7`: `22`
- `index_8`: `25`
- `index_9`: `30`
- *(showing 10 of 29 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(30,)`
- **Data Type:** `uint64`
- **Size:** `30` elements
- **Chunks:** `(30,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `5`
- `index_2`: `10`
- `index_3`: `13`
- `index_4`: `18`
- `index_5`: `25`
- `index_6`: `28`
- `index_7`: `35`
- `index_8`: `39`
- `index_9`: `43`
- *(showing 10 of 30 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(43,)`
- **Data Type:** `uint32`
- **Size:** `43` elements
- **Chunks:** `(43,)`

**Data (Key-Value Format):**

- `index_0`: `500005`
- `index_1`: `500008`
- `index_2`: `500005`
- `index_3`: `500006`
- `index_4`: `500008`
- `index_5`: `500008`
- `index_6`: `500004`
- `index_7`: `500004`
- `index_8`: `500009`
- `index_9`: `500007`
- *(showing 10 of 43 rows using 'uniform' sampling)*


#### Group: /Projections/BC/MC


##### Group: /Projections/BC/MC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(1041,)`
- **Data Type:** `float32`
- **Size:** `1041` elements

**Data (Key-Value Format):**

- `index_0`: `371.73291015625`
- `index_1`: `1484.38037109375`
- `index_2`: `308.739990234375`
- `index_3`: `553.8942260742188`
- `index_4`: `708.0271606445312`
- `index_5`: `158.5495147705078`
- `index_6`: `1234.3226318359375`
- `index_7`: `896.3156127929688`
- `index_8`: `1121.1090087890625`
- `index_9`: `1617.349609375`
- *(showing 10 of 1041 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(1041,)`
- **Data Type:** `float32`
- **Size:** `1041` elements

**Data (Key-Value Format):**

- `index_0`: `1.0`
- `index_1`: `1.0`
- `index_2`: `1.0`
- `index_3`: `1032.35400390625`
- `index_4`: `229.659912109375`
- `index_5`: `812.8301391601562`
- `index_6`: `353.9458312988281`
- `index_7`: `332.3805847167969`
- `index_8`: `190.03729248046875`
- `index_9`: `707.0059814453125`
- *(showing 10 of 1041 rows using 'uniform' sampling)*


##### Group: /Projections/BC/MC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(55,)`
- **Data Type:** `uint32`
- **Size:** `55` elements
- **Chunks:** `(55,)`

**Data (Key-Value Format):**

- `index_0`: `531`
- `index_1`: `595`
- `index_2`: `644`
- `index_3`: `678`
- `index_4`: `803`
- `index_5`: `2263`
- `index_6`: `2305`
- `index_7`: `2333`
- `index_8`: `2375`
- `index_9`: `3536`
- *(showing 10 of 55 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(56,)`
- **Data Type:** `uint64`
- **Size:** `56` elements
- **Chunks:** `(56,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `7`
- `index_2`: `14`
- `index_3`: `23`
- `index_4`: `30`
- `index_5`: `36`
- `index_6`: `44`
- `index_7`: `50`
- `index_8`: `57`
- `index_9`: `66`
- *(showing 10 of 56 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(66,)`
- **Data Type:** `uint64`
- **Size:** `66` elements
- **Chunks:** `(66,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `111`
- `index_2`: `208`
- `index_3`: `324`
- `index_4`: `448`
- `index_5`: `583`
- `index_6`: `692`
- `index_7`: `804`
- `index_8`: `918`
- `index_9`: `1041`
- *(showing 10 of 66 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(1041,)`
- **Data Type:** `uint32`
- **Size:** `1041` elements
- **Chunks:** `(1041,)`

**Data (Key-Value Format):**

- `index_0`: `633`
- `index_1`: `4176`
- `index_2`: `8176`
- `index_3`: `6095`
- `index_4`: `11744`
- `index_5`: `6500`
- `index_6`: `3699`
- `index_7`: `4304`
- `index_8`: `7450`
- `index_9`: `9032`
- *(showing 10 of 1041 rows using 'uniform' sampling)*


#### Group: /Projections/BC/NGFC


##### Group: /Projections/BC/NGFC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(1336,)`
- **Data Type:** `float32`
- **Size:** `1336` elements

**Data (Key-Value Format):**

- `index_0`: `98.53643798828125`
- `index_1`: `396.55078125`
- `index_2`: `51.230770111083984`
- `index_3`: `77.89698028564453`
- `index_4`: `356.3747253417969`
- `index_5`: `560.3941040039062`
- `index_6`: `188.87783813476562`
- `index_7`: `113.68052673339844`
- `index_8`: `148.3682403564453`
- `index_9`: `44.21736526489258`
- *(showing 10 of 1336 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(1336,)`
- **Data Type:** `float32`
- **Size:** `1336` elements

**Data (Key-Value Format):**

- `index_0`: `33.132633209228516`
- `index_1`: `11.513300895690918`
- `index_2`: `1055.7152099609375`
- `index_3`: `158.2823028564453`
- `index_4`: `194.8161163330078`
- `index_5`: `851.980712890625`
- `index_6`: `856.816162109375`
- `index_7`: `19.57091522216797`
- `index_8`: `321.76593017578125`
- `index_9`: `42.41599655151367`
- *(showing 10 of 1336 rows using 'uniform' sampling)*


##### Group: /Projections/BC/NGFC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(55,)`
- **Data Type:** `uint32`
- **Size:** `55` elements
- **Chunks:** `(55,)`

**Data (Key-Value Format):**

- `index_0`: `531`
- `index_1`: `595`
- `index_2`: `644`
- `index_3`: `678`
- `index_4`: `803`
- `index_5`: `2263`
- `index_6`: `2305`
- `index_7`: `2333`
- `index_8`: `2375`
- `index_9`: `3536`
- *(showing 10 of 55 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(56,)`
- **Data Type:** `uint64`
- **Size:** `56` elements
- **Chunks:** `(56,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `7`
- `index_2`: `14`
- `index_3`: `23`
- `index_4`: `30`
- `index_5`: `36`
- `index_6`: `44`
- `index_7`: `50`
- `index_8`: `57`
- `index_9`: `66`
- *(showing 10 of 56 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(66,)`
- **Data Type:** `uint64`
- **Size:** `66` elements
- **Chunks:** `(66,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `107`
- `index_2`: `289`
- `index_3`: `445`
- `index_4`: `626`
- `index_5`: `693`
- `index_6`: `835`
- `index_7`: `988`
- `index_8`: `1172`
- `index_9`: `1336`
- *(showing 10 of 66 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(1336,)`
- **Data Type:** `uint32`
- **Size:** `1336` elements
- **Chunks:** `(1336,)`

**Data (Key-Value Format):**

- `index_0`: `1423`
- `index_1`: `1661`
- `index_2`: `1755`
- `index_3`: `1714`
- `index_4`: `2011`
- `index_5`: `2002`
- `index_6`: `2289`
- `index_7`: `1775`
- `index_8`: `2075`
- `index_9`: `2061`
- *(showing 10 of 1336 rows using 'uniform' sampling)*


### Group: /Projections/GC

#### Attributes

- **ahdf5-smd-AAC:** `BEST GUESS: GC→AAC projection: Granule Cell excitatory synapses onto Axo-Axonic Cell targets. Contains 46 edges; source GIDs range 54–418.` (type: `str`)
- **ahdf5-smd-BC:** `BEST GUESS: GC→BC projection: Granule Cell excitatory synapses onto Basket Cell targets.` (type: `str`)
- **ahdf5-smd-HC:** `BEST GUESS: GC→HC projection: Granule Cell excitatory synapses onto HIPP Cell (Hilar Commissural-Associational Pathway cell) targets.` (type: `str`)
- **ahdf5-smd-MC:** `BEST GUESS: GC→MC projection: Granule Cell excitatory synapses onto Mossy Cell targets via mossy fibre collaterals.` (type: `str`)
- **ahdf5-smd-NGFC:** `BEST GUESS: GC→NGFC projection: Granule Cell excitatory synapses onto Neurogliaform Cell targets.` (type: `str`)


#### Group: /Projections/GC/AAC

##### Attributes

- **ahdf5-smd-Attributes:** `BEST GUESS: Per-edge spatial distance attributes (Longitudinal, Transverse) for the GC→AAC projection.` (type: `str`)
- **ahdf5-smd-Edges:** `BEST GUESS: Destination Block Sparse connectivity arrays for the GC→AAC projection (46 edges).` (type: `str`)


##### Group: /Projections/GC/AAC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(46,)`
- **Data Type:** `float32`
- **Size:** `46` elements

**Data (Key-Value Format):**

- `index_0`: `136.30575561523438`
- `index_1`: `135.41864013671875`
- `index_2`: `86.88957214355469`
- `index_3`: `46.56898880004883`
- `index_4`: `135.41864013671875`
- `index_5`: `268.3645324707031`
- `index_6`: `312.16607666015625`
- `index_7`: `121.29556274414062`
- `index_8`: `100.36931610107422`
- `index_9`: `312.16607666015625`
- *(showing 10 of 46 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(46,)`
- **Data Type:** `float32`
- **Size:** `46` elements

**Data (Key-Value Format):**

- `index_0`: `282.26904296875`
- `index_1`: `78.59910583496094`
- `index_2`: `287.8413391113281`
- `index_3`: `2.859376907348633`
- `index_4`: `78.59910583496094`
- `index_5`: `352.366455078125`
- `index_6`: `442.66864013671875`
- `index_7`: `137.96517944335938`
- `index_8`: `7.418144226074219`
- `index_9`: `442.66864013671875`
- *(showing 10 of 46 rows using 'uniform' sampling)*


##### Group: /Projections/GC/AAC/Edges

###### Attributes

- **ahdf5-smd-Destination Block Index:** `BEST GUESS: First destination GID of each contiguous block in the GC→AAC DBS representation.` (type: `str`)
- **ahdf5-smd-Destination Block Pointer:** `BEST GUESS: Offsets into Destination Pointer marking block starts for the GC→AAC projection.` (type: `str`)
- **ahdf5-smd-Destination Pointer:** `BEST GUESS: Offsets into Source Index for each AAC destination in the GC→AAC projection.` (type: `str`)
- **ahdf5-smd-Source Index:** `BEST GUESS: Flat array of source GC GIDs for all 46 edges in the GC→AAC projection (uint32, range 54–418).` (type: `str`)


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(1,)`
- **Data Type:** `uint32`
- **Size:** `1` elements
- **Chunks:** `(1,)`

**Data (Key-Value Format):**

- `index_0`: `500000`


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(2,)`
- **Data Type:** `uint64`
- **Size:** `2` elements
- **Chunks:** `(2,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `10`


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(10,)`
- **Data Type:** `uint64`
- **Size:** `10` elements
- **Chunks:** `(10,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `10`
- `index_2`: `13`
- `index_3`: `18`
- `index_4`: `21`
- `index_5`: `24`
- `index_6`: `30`
- `index_7`: `35`
- `index_8`: `42`
- `index_9`: `46`


###### Dataset: Source Index

####### Properties

- **Shape:** `(46,)`
- **Data Type:** `uint32`
- **Size:** `46` elements
- **Chunks:** `(46,)`

**Data (Key-Value Format):**

- `index_0`: `69`
- `index_1`: `279`
- `index_2`: `80`
- `index_3`: `78`
- `index_4`: `277`
- `index_5`: `69`
- `index_6`: `72`
- `index_7`: `79`
- `index_8`: `280`
- `index_9`: `281`
- *(showing 10 of 46 rows using 'uniform' sampling)*


#### Group: /Projections/GC/BC


##### Group: /Projections/GC/BC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(70,)`
- **Data Type:** `float32`
- **Size:** `70` elements

**Data (Key-Value Format):**

- `index_0`: `223.69732666015625`
- `index_1`: `223.69732666015625`
- `index_2`: `133.380615234375`
- `index_3`: `463.9686279296875`
- `index_4`: `321.7530212402344`
- `index_5`: `510.1578063964844`
- `index_6`: `133.380615234375`
- `index_7`: `200.44068908691406`
- `index_8`: `463.9686279296875`
- `index_9`: `186.2488250732422`
- *(showing 10 of 70 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(70,)`
- **Data Type:** `float32`
- **Size:** `70` elements

**Data (Key-Value Format):**

- `index_0`: `89.04246520996094`
- `index_1`: `89.04246520996094`
- `index_2`: `44.63631057739258`
- `index_3`: `161.66232299804688`
- `index_4`: `59.429439544677734`
- `index_5`: `93.61544036865234`
- `index_6`: `44.63631057739258`
- `index_7`: `37.34847640991211`
- `index_8`: `161.66232299804688`
- `index_9`: `299.3685302734375`
- *(showing 10 of 70 rows using 'uniform' sampling)*


##### Group: /Projections/GC/BC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(1,)`
- **Data Type:** `uint32`
- **Size:** `1` elements
- **Chunks:** `(1,)`

**Data (Key-Value Format):**

- `index_0`: `500000`


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(2,)`
- **Data Type:** `uint64`
- **Size:** `2` elements
- **Chunks:** `(2,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `11`


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(11,)`
- **Data Type:** `uint64`
- **Size:** `11` elements
- **Chunks:** `(11,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `9`
- `index_2`: `15`
- `index_3`: `24`
- `index_4`: `29`
- `index_5`: `42`
- `index_6`: `49`
- `index_7`: `57`
- `index_8`: `59`
- `index_9`: `70`
- *(showing 10 of 11 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(70,)`
- **Data Type:** `uint32`
- **Size:** `70` elements
- **Chunks:** `(70,)`

**Data (Key-Value Format):**

- `index_0`: `568`
- `index_1`: `2364`
- `index_2`: `580`
- `index_3`: `2370`
- `index_4`: `660`
- `index_5`: `2406`
- `index_6`: `694`
- `index_7`: `2290`
- `index_8`: `646`
- `index_9`: `2375`
- *(showing 10 of 70 rows using 'uniform' sampling)*


#### Group: /Projections/GC/HC


##### Group: /Projections/GC/HC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(96,)`
- **Data Type:** `float32`
- **Size:** `96` elements

**Data (Key-Value Format):**

- `index_0`: `252.6704559326172`
- `index_1`: `81.62577056884766`
- `index_2`: `252.6704559326172`
- `index_3`: `157.301513671875`
- `index_4`: `267.7645263671875`
- `index_5`: `267.7645263671875`
- `index_6`: `527.1551513671875`
- `index_7`: `181.83734130859375`
- `index_8`: `733.7763061523438`
- `index_9`: `1527.9168701171875`
- *(showing 10 of 96 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(96,)`
- **Data Type:** `float32`
- **Size:** `96` elements

**Data (Key-Value Format):**

- `index_0`: `99.80271911621094`
- `index_1`: `129.75975036621094`
- `index_2`: `99.80271911621094`
- `index_3`: `188.9913787841797`
- `index_4`: `180.95484924316406`
- `index_5`: `180.95484924316406`
- `index_6`: `130.2235870361328`
- `index_7`: `763.1771240234375`
- `index_8`: `160.69979858398438`
- `index_9`: `4.305202960968018`
- *(showing 10 of 96 rows using 'uniform' sampling)*


##### Group: /Projections/GC/HC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(1,)`
- **Data Type:** `uint32`
- **Size:** `1` elements
- **Chunks:** `(1,)`

**Data (Key-Value Format):**

- `index_0`: `500000`


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(2,)`
- **Data Type:** `uint64`
- **Size:** `2` elements
- **Chunks:** `(2,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `11`


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(11,)`
- **Data Type:** `uint64`
- **Size:** `11` elements
- **Chunks:** `(11,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `14`
- `index_2`: `21`
- `index_3`: `36`
- `index_4`: `50`
- `index_5`: `56`
- `index_6`: `68`
- `index_7`: `77`
- `index_8`: `84`
- `index_9`: `96`
- *(showing 10 of 11 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(96,)`
- **Data Type:** `uint32`
- **Size:** `96` elements
- **Chunks:** `(96,)`

**Data (Key-Value Format):**

- `index_0`: `1618`
- `index_1`: `2812`
- `index_2`: `774`
- `index_3`: `2954`
- `index_4`: `3000`
- `index_5`: `1521`
- `index_6`: `2676`
- `index_7`: `2565`
- `index_8`: `1024`
- `index_9`: `2118`
- *(showing 10 of 96 rows using 'uniform' sampling)*


#### Group: /Projections/GC/MC


##### Group: /Projections/GC/MC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(6425,)`
- **Data Type:** `float32`
- **Size:** `6425` elements

**Data (Key-Value Format):**

- `index_0`: `46.001953125`
- `index_1`: `998.554443359375`
- `index_2`: `1828.512451171875`
- `index_3`: `2401.1044921875`
- `index_4`: `1462.3839111328125`
- `index_5`: `198.3531494140625`
- `index_6`: `1376.9002685546875`
- `index_7`: `1610.576904296875`
- `index_8`: `2585.281005859375`
- `index_9`: `289.55126953125`
- *(showing 10 of 6425 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(6425,)`
- **Data Type:** `float32`
- **Size:** `6425` elements

**Data (Key-Value Format):**

- `index_0`: `1.0`
- `index_1`: `1.0`
- `index_2`: `1.0`
- `index_3`: `1.0`
- `index_4`: `1.0`
- `index_5`: `1.0`
- `index_6`: `1.0`
- `index_7`: `1.0`
- `index_8`: `1.0`
- `index_9`: `1.0`
- *(showing 10 of 6425 rows using 'uniform' sampling)*


##### Group: /Projections/GC/MC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(1,)`
- **Data Type:** `uint32`
- **Size:** `1` elements
- **Chunks:** `(1,)`

**Data (Key-Value Format):**

- `index_0`: `500000`


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(2,)`
- **Data Type:** `uint64`
- **Size:** `2` elements
- **Chunks:** `(2,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `11`


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(11,)`
- **Data Type:** `uint64`
- **Size:** `11` elements
- **Chunks:** `(11,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `456`
- `index_2`: `2241`
- `index_3`: `2787`
- `index_4`: `3417`
- `index_5`: `3439`
- `index_6`: `4652`
- `index_7`: `5212`
- `index_8`: `5781`
- `index_9`: `6425`
- *(showing 10 of 11 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(6425,)`
- **Data Type:** `uint32`
- **Size:** `6425` elements
- **Chunks:** `(4096,)`

**Data (Key-Value Format):**

- `index_0`: `4507`
- `index_1`: `4116`
- `index_2`: `8072`
- `index_3`: `13505`
- `index_4`: `3137`
- `index_5`: `2881`
- `index_6`: `8229`
- `index_7`: `7094`
- `index_8`: `10044`
- `index_9`: `2714`
- *(showing 10 of 6425 rows using 'uniform' sampling)*


#### Group: /Projections/GC/NGFC


##### Group: /Projections/GC/NGFC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(240,)`
- **Data Type:** `float32`
- **Size:** `240` elements

**Data (Key-Value Format):**

- `index_0`: `439.5881042480469`
- `index_1`: `47.44321060180664`
- `index_2`: `71.19010162353516`
- `index_3`: `198.35888671875`
- `index_4`: `0.4944570064544678`
- `index_5`: `54.55183029174805`
- `index_6`: `454.6945495605469`
- `index_7`: `149.51364135742188`
- `index_8`: `149.51364135742188`
- `index_9`: `55.549259185791016`
- *(showing 10 of 240 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(240,)`
- **Data Type:** `float32`
- **Size:** `240` elements

**Data (Key-Value Format):**

- `index_0`: `36.59138107299805`
- `index_1`: `558.7125854492188`
- `index_2`: `384.7575988769531`
- `index_3`: `585.7324829101562`
- `index_4`: `533.3656616210938`
- `index_5`: `103.34819793701172`
- `index_6`: `205.95333862304688`
- `index_7`: `181.60598754882812`
- `index_8`: `181.60598754882812`
- `index_9`: `263.1370544433594`
- *(showing 10 of 240 rows using 'uniform' sampling)*


##### Group: /Projections/GC/NGFC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(1,)`
- **Data Type:** `uint32`
- **Size:** `1` elements
- **Chunks:** `(1,)`

**Data (Key-Value Format):**

- `index_0`: `500000`


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(2,)`
- **Data Type:** `uint64`
- **Size:** `2` elements
- **Chunks:** `(2,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `11`


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(11,)`
- **Data Type:** `uint64`
- **Size:** `11` elements
- **Chunks:** `(11,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `52`
- `index_2`: `72`
- `index_3`: `99`
- `index_4`: `119`
- `index_5`: `137`
- `index_6`: `157`
- `index_7`: `180`
- `index_8`: `199`
- `index_9`: `240`
- *(showing 10 of 11 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(240,)`
- **Data Type:** `uint32`
- **Size:** `240` elements
- **Chunks:** `(240,)`

**Data (Key-Value Format):**

- `index_0`: `1655`
- `index_1`: `1924`
- `index_2`: `1712`
- `index_3`: `1724`
- `index_4`: `1783`
- `index_5`: `1894`
- `index_6`: `1669`
- `index_7`: `1683`
- `index_8`: `1915`
- `index_9`: `2075`
- *(showing 10 of 240 rows using 'uniform' sampling)*


### Group: /Projections/HC


#### Group: /Projections/HC/HC


##### Group: /Projections/HC/HC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(27,)`
- **Data Type:** `float32`
- **Size:** `27` elements

**Data (Key-Value Format):**

- `index_0`: `701.8274536132812`
- `index_1`: `107.89092254638672`
- `index_2`: `107.89092254638672`
- `index_3`: `32.00617980957031`
- `index_4`: `245.93470764160156`
- `index_5`: `632.55322265625`
- `index_6`: `245.93470764160156`
- `index_7`: `4.620658874511719`
- `index_8`: `481.4839172363281`
- `index_9`: `632.55322265625`
- *(showing 10 of 27 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(27,)`
- **Data Type:** `float32`
- **Size:** `27` elements

**Data (Key-Value Format):**

- `index_0`: `852.6820068359375`
- `index_1`: `15.394277572631836`
- `index_2`: `15.394277572631836`
- `index_3`: `55.590576171875`
- `index_4`: `44.692413330078125`
- `index_5`: `80.9241943359375`
- `index_6`: `44.692413330078125`
- `index_7`: `5.052164077758789`
- `index_8`: `207.4703369140625`
- `index_9`: `80.9241943359375`
- *(showing 10 of 27 rows using 'uniform' sampling)*


##### Group: /Projections/HC/HC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(24,)`
- **Data Type:** `uint32`
- **Size:** `24` elements
- **Chunks:** `(24,)`

**Data (Key-Value Format):**

- `index_0`: `1148`
- `index_1`: `1317`
- `index_2`: `1736`
- `index_3`: `1816`
- `index_4`: `2229`
- `index_5`: `2329`
- `index_6`: `2494`
- `index_7`: `2646`
- `index_8`: `2896`
- `index_9`: `3881`
- *(showing 10 of 24 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(25,)`
- **Data Type:** `uint64`
- **Size:** `25` elements
- **Chunks:** `(25,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `2`
- `index_2`: `5`
- `index_3`: `8`
- `index_4`: `10`
- `index_5`: `13`
- `index_6`: `16`
- `index_7`: `18`
- `index_8`: `21`
- `index_9`: `25`
- *(showing 10 of 25 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(25,)`
- **Data Type:** `uint64`
- **Size:** `25` elements
- **Chunks:** `(25,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `3`
- `index_2`: `7`
- `index_3`: `10`
- `index_4`: `12`
- `index_5`: `16`
- `index_6`: `19`
- `index_7`: `21`
- `index_8`: `24`
- `index_9`: `27`
- *(showing 10 of 25 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(27,)`
- **Data Type:** `uint32`
- **Size:** `27` elements
- **Chunks:** `(27,)`

**Data (Key-Value Format):**

- `index_0`: `1550`
- `index_1`: `1624`
- `index_2`: `1651`
- `index_3`: `1792`
- `index_4`: `2417`
- `index_5`: `2525`
- `index_6`: `2530`
- `index_7`: `2333`
- `index_8`: `2220`
- `index_9`: `1698`
- *(showing 10 of 27 rows using 'uniform' sampling)*


#### Group: /Projections/HC/MC


##### Group: /Projections/HC/MC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(64274,)`
- **Data Type:** `float32`
- **Size:** `64274` elements

**Data (Key-Value Format):**

- `index_0`: `149.56253051757812`
- `index_1`: `441.8834228515625`
- `index_2`: `1149.03173828125`
- `index_3`: `548.813720703125`
- `index_4`: `59.12883758544922`
- `index_5`: `221.47007751464844`
- `index_6`: `1628.8765869140625`
- `index_7`: `2591.0888671875`
- `index_8`: `896.1787109375`
- `index_9`: `616.3333129882812`
- *(showing 10 of 64274 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(64274,)`
- **Data Type:** `float32`
- **Size:** `64274` elements

**Data (Key-Value Format):**

- `index_0`: `1.0`
- `index_1`: `1.0`
- `index_2`: `1.0`
- `index_3`: `1.0`
- `index_4`: `1.0`
- `index_5`: `1.0`
- `index_6`: `1.0`
- `index_7`: `1.0`
- `index_8`: `1.0`
- `index_9`: `1.0`
- *(showing 10 of 64274 rows using 'uniform' sampling)*


##### Group: /Projections/HC/MC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(92,)`
- **Data Type:** `uint32`
- **Size:** `92` elements
- **Chunks:** `(92,)`

**Data (Key-Value Format):**

- `index_0`: `774`
- `index_1`: `1521`
- `index_2`: `1736`
- `index_3`: `2118`
- `index_4`: `2327`
- `index_5`: `2530`
- `index_6`: `2693`
- `index_7`: `2954`
- `index_8`: `3448`
- `index_9`: `5296`
- *(showing 10 of 92 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(93,)`
- **Data Type:** `uint64`
- **Size:** `93` elements
- **Chunks:** `(93,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `11`
- `index_2`: `21`
- `index_3`: `31`
- `index_4`: `41`
- `index_5`: `53`
- `index_6`: `63`
- `index_7`: `73`
- `index_8`: `83`
- `index_9`: `95`
- *(showing 10 of 93 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(95,)`
- **Data Type:** `uint64`
- **Size:** `95` elements
- **Chunks:** `(95,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `4926`
- `index_2`: `12948`
- `index_3`: `21076`
- `index_4`: `29535`
- `index_5`: `37268`
- `index_6`: `43116`
- `index_7`: `51902`
- `index_8`: `59490`
- `index_9`: `64274`
- *(showing 10 of 95 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(64274,)`
- **Data Type:** `uint32`
- **Size:** `64274` elements
- **Chunks:** `(4096,)`

**Data (Key-Value Format):**

- `index_0`: `93`
- `index_1`: `5915`
- `index_2`: `2350`
- `index_3`: `4406`
- `index_4`: `5396`
- `index_5`: `2815`
- `index_6`: `8854`
- `index_7`: `9058`
- `index_8`: `7711`
- `index_9`: `29675`
- *(showing 10 of 64274 rows using 'uniform' sampling)*


### Group: /Projections/MC


#### Group: /Projections/MC/AAC


##### Group: /Projections/MC/AAC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(16036,)`
- **Data Type:** `float32`
- **Size:** `16036` elements

**Data (Key-Value Format):**

- `index_0`: `131.21641540527344`
- `index_1`: `157.33203125`
- `index_2`: `157.33203125`
- `index_3`: `131.21641540527344`
- `index_4`: `274.8634338378906`
- `index_5`: `274.8634338378906`
- `index_6`: `1031.7979736328125`
- `index_7`: `1031.7979736328125`
- `index_8`: `906.3486938476562`
- `index_9`: `499.93798828125`
- *(showing 10 of 16036 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(16036,)`
- **Data Type:** `float32`
- **Size:** `16036` elements

**Data (Key-Value Format):**

- `index_0`: `259.7137451171875`
- `index_1`: `454.1041259765625`
- `index_2`: `454.1041259765625`
- `index_3`: `259.7137451171875`
- `index_4`: `229.90359497070312`
- `index_5`: `229.90359497070312`
- `index_6`: `70.06658172607422`
- `index_7`: `70.06658172607422`
- `index_8`: `224.382568359375`
- `index_9`: `182.59613037109375`
- *(showing 10 of 16036 rows using 'uniform' sampling)*


##### Group: /Projections/MC/AAC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(1227,)`
- **Data Type:** `uint32`
- **Size:** `1227` elements
- **Chunks:** `(1227,)`

**Data (Key-Value Format):**

- `index_0`: `1972`
- `index_1`: `4487`
- `index_2`: `5119`
- `index_3`: `5822`
- `index_4`: `6414`
- `index_5`: `6982`
- `index_6`: `7537`
- `index_7`: `8162`
- `index_8`: `8788`
- `index_9`: `11703`
- *(showing 10 of 1227 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(1228,)`
- **Data Type:** `uint64`
- **Size:** `1228` elements
- **Chunks:** `(1228,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `193`
- `index_2`: `376`
- `index_3`: `559`
- `index_4`: `786`
- `index_5`: `1034`
- `index_6`: `1287`
- `index_7`: `1516`
- `index_8`: `1718`
- `index_9`: `1916`
- *(showing 10 of 1228 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(1916,)`
- **Data Type:** `uint64`
- **Size:** `1916` elements
- **Chunks:** `(1916,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `1196`
- `index_2`: `2278`
- `index_3`: `3456`
- `index_4`: `6058`
- `index_5`: `8736`
- `index_6`: `11378`
- `index_7`: `13350`
- `index_8`: `14770`
- `index_9`: `16036`
- *(showing 10 of 1916 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(16036,)`
- **Data Type:** `uint32`
- **Size:** `16036` elements
- **Chunks:** `(4096,)`

**Data (Key-Value Format):**

- `index_0`: `67`
- `index_1`: `280`
- `index_2`: `58`
- `index_3`: `280`
- `index_4`: `278`
- `index_5`: `280`
- `index_6`: `266`
- `index_7`: `79`
- `index_8`: `79`
- `index_9`: `295`
- *(showing 10 of 16036 rows using 'uniform' sampling)*


#### Group: /Projections/MC/BC


##### Group: /Projections/MC/BC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(12754,)`
- **Data Type:** `float32`
- **Size:** `12754` elements

**Data (Key-Value Format):**

- `index_0`: `114.00726318359375`
- `index_1`: `144.67930603027344`
- `index_2`: `549.7229614257812`
- `index_3`: `279.6559753417969`
- `index_4`: `161.44041442871094`
- `index_5`: `5.398026943206787`
- `index_6`: `158.13949584960938`
- `index_7`: `47.034358978271484`
- `index_8`: `532.7432861328125`
- `index_9`: `70.52981567382812`
- *(showing 10 of 12754 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(12754,)`
- **Data Type:** `float32`
- **Size:** `12754` elements

**Data (Key-Value Format):**

- `index_0`: `84.3740463256836`
- `index_1`: `328.54833984375`
- `index_2`: `170.37786865234375`
- `index_3`: `506.00982666015625`
- `index_4`: `173.1712188720703`
- `index_5`: `370.1376953125`
- `index_6`: `257.7042541503906`
- `index_7`: `10.958274841308594`
- `index_8`: `153.6866912841797`
- `index_9`: `58.95123291015625`
- *(showing 10 of 12754 rows using 'uniform' sampling)*


##### Group: /Projections/MC/BC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(1073,)`
- **Data Type:** `uint32`
- **Size:** `1073` elements
- **Chunks:** `(1073,)`

**Data (Key-Value Format):**

- `index_0`: `1867`
- `index_1`: `4393`
- `index_2`: `5051`
- `index_3`: `5686`
- `index_4`: `6286`
- `index_5`: `6774`
- `index_6`: `7266`
- `index_7`: `7816`
- `index_8`: `8537`
- `index_9`: `11733`
- *(showing 10 of 1073 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(1074,)`
- **Data Type:** `uint64`
- **Size:** `1074` elements
- **Chunks:** `(1074,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `155`
- `index_2`: `311`
- `index_3`: `462`
- `index_4`: `647`
- `index_5`: `855`
- `index_6`: `1044`
- `index_7`: `1267`
- `index_8`: `1415`
- `index_9`: `1582`
- *(showing 10 of 1074 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(1582,)`
- **Data Type:** `uint64`
- **Size:** `1582` elements
- **Chunks:** `(1582,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `1054`
- `index_2`: `2018`
- `index_3`: `3070`
- `index_4`: `4848`
- `index_5`: `7058`
- `index_6`: `8864`
- `index_7`: `11010`
- `index_8`: `11944`
- `index_9`: `12754`
- *(showing 10 of 1582 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(12754,)`
- **Data Type:** `uint32`
- **Size:** `12754` elements
- **Chunks:** `(4096,)`

**Data (Key-Value Format):**

- `index_0`: `2193`
- `index_1`: `539`
- `index_2`: `2330`
- `index_3`: `2380`
- `index_4`: `2407`
- `index_5`: `606`
- `index_6`: `2369`
- `index_7`: `609`
- `index_8`: `735`
- `index_9`: `735`
- *(showing 10 of 12754 rows using 'uniform' sampling)*


#### Group: /Projections/MC/GC


##### Group: /Projections/MC/GC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(82,)`
- **Data Type:** `float32`
- **Size:** `82` elements

**Data (Key-Value Format):**

- `index_0`: `96.75601196289062`
- `index_1`: `173.329833984375`
- `index_2`: `47.16160202026367`
- `index_3`: `51.83583450317383`
- `index_4`: `68.13371276855469`
- `index_5`: `370.48052978515625`
- `index_6`: `307.8371276855469`
- `index_7`: `191.174560546875`
- `index_8`: `44.708892822265625`
- `index_9`: `37.36309814453125`
- *(showing 10 of 82 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(82,)`
- **Data Type:** `float32`
- **Size:** `82` elements

**Data (Key-Value Format):**

- `index_0`: `139.73568725585938`
- `index_1`: `115.5565185546875`
- `index_2`: `242.8964080810547`
- `index_3`: `160.85240173339844`
- `index_4`: `222.2632293701172`
- `index_5`: `148.19680786132812`
- `index_6`: `228.92457580566406`
- `index_7`: `243.3870849609375`
- `index_8`: `168.4822998046875`
- `index_9`: `126.19596862792969`
- *(showing 10 of 82 rows using 'uniform' sampling)*


##### Group: /Projections/MC/GC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(40,)`
- **Data Type:** `uint32`
- **Size:** `40` elements
- **Chunks:** `(40,)`

**Data (Key-Value Format):**

- `index_0`: `6002`
- `index_1`: `6307`
- `index_2`: `6432`
- `index_3`: `6844`
- `index_4`: `7146`
- `index_5`: `7321`
- `index_6`: `7443`
- `index_7`: `7601`
- `index_8`: `7849`
- `index_9`: `9176`
- *(showing 10 of 40 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(41,)`
- **Data Type:** `uint64`
- **Size:** `41` elements
- **Chunks:** `(41,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `4`
- `index_2`: `8`
- `index_3`: `13`
- `index_4`: `17`
- `index_5`: `22`
- `index_6`: `26`
- `index_7`: `31`
- `index_8`: `35`
- `index_9`: `41`
- *(showing 10 of 41 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(41,)`
- **Data Type:** `uint64`
- **Size:** `41` elements
- **Chunks:** `(41,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `8`
- `index_2`: `16`
- `index_3`: `26`
- `index_4`: `34`
- `index_5`: `44`
- `index_6`: `52`
- `index_7`: `62`
- `index_8`: `72`
- `index_9`: `82`
- *(showing 10 of 41 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(82,)`
- **Data Type:** `uint32`
- **Size:** `82` elements
- **Chunks:** `(82,)`

**Data (Key-Value Format):**

- `index_0`: `500007`
- `index_1`: `500001`
- `index_2`: `500006`
- `index_3`: `500001`
- `index_4`: `500003`
- `index_5`: `500000`
- `index_6`: `500005`
- `index_7`: `500002`
- `index_8`: `500003`
- `index_9`: `500001`
- *(showing 10 of 82 rows using 'uniform' sampling)*


#### Group: /Projections/MC/HC


##### Group: /Projections/MC/HC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(2570,)`
- **Data Type:** `float32`
- **Size:** `2570` elements

**Data (Key-Value Format):**

- `index_0`: `31.61420249938965`
- `index_1`: `12.037714004516602`
- `index_2`: `327.0094909667969`
- `index_3`: `270.03472900390625`
- `index_4`: `1979.9825439453125`
- `index_5`: `327.0094909667969`
- `index_6`: `31.61420249938965`
- `index_7`: `89.76390838623047`
- `index_8`: `772.0818481445312`
- `index_9`: `1485.374267578125`
- *(showing 10 of 2570 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(2570,)`
- **Data Type:** `float32`
- **Size:** `2570` elements

**Data (Key-Value Format):**

- `index_0`: `161.2295684814453`
- `index_1`: `419.1921081542969`
- `index_2`: `5.505444049835205`
- `index_3`: `153.87637329101562`
- `index_4`: `459.95196533203125`
- `index_5`: `5.505444049835205`
- `index_6`: `161.2295684814453`
- `index_7`: `218.14817810058594`
- `index_8`: `464.69268798828125`
- `index_9`: `157.00299072265625`
- *(showing 10 of 2570 rows using 'uniform' sampling)*


##### Group: /Projections/MC/HC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(935,)`
- **Data Type:** `uint32`
- **Size:** `935` elements
- **Chunks:** `(935,)`

**Data (Key-Value Format):**

- `index_0`: `762`
- `index_1`: `3266`
- `index_2`: `4405`
- `index_3`: `5249`
- `index_4`: `6153`
- `index_5`: `6885`
- `index_6`: `7677`
- `index_7`: `8621`
- `index_8`: `9995`
- `index_9`: `18202`
- *(showing 10 of 935 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(936,)`
- **Data Type:** `uint64`
- **Size:** `936` elements
- **Chunks:** `(936,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `109`
- `index_2`: `219`
- `index_3`: `344`
- `index_4`: `463`
- `index_5`: `591`
- `index_6`: `714`
- `index_7`: `829`
- `index_8`: `948`
- `index_9`: `1055`
- *(showing 10 of 936 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(1055,)`
- **Data Type:** `uint64`
- **Size:** `1055` elements
- **Chunks:** `(1055,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `260`
- `index_2`: `552`
- `index_3`: `830`
- `index_4`: `1106`
- `index_5`: `1404`
- `index_6`: `1726`
- `index_7`: `2016`
- `index_8`: `2308`
- `index_9`: `2570`
- *(showing 10 of 1055 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(2570,)`
- **Data Type:** `uint32`
- **Size:** `2570` elements
- **Chunks:** `(2570,)`

**Data (Key-Value Format):**

- `index_0`: `1850`
- `index_1`: `1005`
- `index_2`: `2333`
- `index_3`: `3114`
- `index_4`: `2954`
- `index_5`: `2674`
- `index_6`: `2417`
- `index_7`: `2812`
- `index_8`: `3438`
- `index_9`: `4180`
- *(showing 10 of 2570 rows using 'uniform' sampling)*


#### Group: /Projections/MC/MC


##### Group: /Projections/MC/MC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(1083050,)`
- **Data Type:** `float32`
- **Size:** `1083050` elements

**Data (Key-Value Format):**

- `index_0`: `75.99513244628906`
- `index_1`: `841.60205078125`
- `index_2`: `805.92822265625`
- `index_3`: `1225.471435546875`
- `index_4`: `870.66943359375`
- `index_5`: `2909.4814453125`
- `index_6`: `924.386962890625`
- `index_7`: `869.3661499023438`
- `index_8`: `3098.04296875`
- `index_9`: `91.52047729492188`
- *(showing 10 of 1083050 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(1083050,)`
- **Data Type:** `float32`
- **Size:** `1083050` elements

**Data (Key-Value Format):**

- `index_0`: `1.0`
- `index_1`: `1.0`
- `index_2`: `1.0`
- `index_3`: `1.0`
- `index_4`: `1.0`
- `index_5`: `1.0`
- `index_6`: `1.0`
- `index_7`: `1.0`
- `index_8`: `1.0`
- `index_9`: `1.0`
- *(showing 10 of 1083050 rows using 'uniform' sampling)*


##### Group: /Projections/MC/MC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(2913,)`
- **Data Type:** `uint32`
- **Size:** `2913` elements
- **Chunks:** `(2913,)`

**Data (Key-Value Format):**

- `index_0`: `59`
- `index_1`: `2554`
- `index_2`: `3993`
- `index_3`: `5276`
- `index_4`: `6533`
- `index_5`: `7822`
- `index_6`: `9151`
- `index_7`: `10604`
- `index_8`: `12514`
- `index_9`: `29675`
- *(showing 10 of 2913 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(2914,)`
- **Data Type:** `uint64`
- **Size:** `2914` elements
- **Chunks:** `(2914,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `441`
- `index_2`: `936`
- `index_3`: `1557`
- `index_4`: `2148`
- `index_5`: `2747`
- `index_6`: `3388`
- `index_7`: `3911`
- `index_8`: `4373`
- `index_9`: `4733`
- *(showing 10 of 2914 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(4733,)`
- **Data Type:** `uint64`
- **Size:** `4733` elements
- **Chunks:** `(4096,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `110560`
- `index_2`: `243584`
- `index_3`: `391934`
- `index_4`: `540068`
- `index_5`: `694032`
- `index_6`: `827372`
- `index_7`: `937808`
- `index_8`: `1028902`
- `index_9`: `1083050`
- *(showing 10 of 4733 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(1083050,)`
- **Data Type:** `uint32`
- **Size:** `1083050` elements
- **Chunks:** `(4096,)`

**Data (Key-Value Format):**

- `index_0`: `59`
- `index_1`: `7358`
- `index_2`: `4235`
- `index_3`: `2899`
- `index_4`: `8854`
- `index_5`: `7549`
- `index_6`: `6618`
- `index_7`: `7433`
- `index_8`: `6668`
- `index_9`: `19158`
- *(showing 10 of 1083050 rows using 'uniform' sampling)*


### Group: /Projections/NGFC


#### Group: /Projections/NGFC/HC


##### Group: /Projections/NGFC/HC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(190,)`
- **Data Type:** `float32`
- **Size:** `190` elements

**Data (Key-Value Format):**

- `index_0`: `214.34146118164062`
- `index_1`: `58.45044708251953`
- `index_2`: `581.649169921875`
- `index_3`: `826.0678100585938`
- `index_4`: `744.7484130859375`
- `index_5`: `344.61328125`
- `index_6`: `880.490234375`
- `index_7`: `344.61328125`
- `index_8`: `1634.991943359375`
- `index_9`: `2133.68505859375`
- *(showing 10 of 190 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(190,)`
- **Data Type:** `float32`
- **Size:** `190` elements

**Data (Key-Value Format):**

- `index_0`: `228.21095275878906`
- `index_1`: `280.6109619140625`
- `index_2`: `210.2220001220703`
- `index_3`: `27.757091522216797`
- `index_4`: `50.701988220214844`
- `index_5`: `29.71853256225586`
- `index_6`: `466.20916748046875`
- `index_7`: `29.71853256225586`
- `index_8`: `522.8258056640625`
- `index_9`: `28.831438064575195`
- *(showing 10 of 190 rows using 'uniform' sampling)*


##### Group: /Projections/NGFC/HC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(81,)`
- **Data Type:** `uint32`
- **Size:** `81` elements
- **Chunks:** `(81,)`

**Data (Key-Value Format):**

- `index_0`: `1207`
- `index_1`: `1669`
- `index_2`: `1737`
- `index_3`: `1800`
- `index_4`: `1846`
- `index_5`: `1881`
- `index_6`: `1966`
- `index_7`: `2027`
- `index_8`: `2177`
- `index_9`: `2425`
- *(showing 10 of 81 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(82,)`
- **Data Type:** `uint64`
- **Size:** `82` elements
- **Chunks:** `(82,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `9`
- `index_2`: `24`
- `index_3`: `38`
- `index_4`: `47`
- `index_5`: `59`
- `index_6`: `76`
- `index_7`: `91`
- `index_8`: `103`
- `index_9`: `113`
- *(showing 10 of 82 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(113,)`
- **Data Type:** `uint64`
- **Size:** `113` elements
- **Chunks:** `(113,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `19`
- `index_2`: `35`
- `index_3`: `52`
- `index_4`: `78`
- `index_5`: `100`
- `index_6`: `123`
- `index_7`: `143`
- `index_8`: `166`
- `index_9`: `190`
- *(showing 10 of 113 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(190,)`
- **Data Type:** `uint32`
- **Size:** `190` elements
- **Chunks:** `(190,)`

**Data (Key-Value Format):**

- `index_0`: `1816`
- `index_1`: `3489`
- `index_2`: `1557`
- `index_3`: `1557`
- `index_4`: `1826`
- `index_5`: `2333`
- `index_6`: `2220`
- `index_7`: `2682`
- `index_8`: `2183`
- `index_9`: `2896`
- *(showing 10 of 190 rows using 'uniform' sampling)*


#### Group: /Projections/NGFC/NGFC


##### Group: /Projections/NGFC/NGFC/Attributes


###### Dataset: Longitudinal

####### Properties

- **Shape:** `(324,)`
- **Data Type:** `float32`
- **Size:** `324` elements

**Data (Key-Value Format):**

- `index_0`: `3.774722099304199`
- `index_1`: `53.529239654541016`
- `index_2`: `53.529239654541016`
- `index_3`: `46.847328186035156`
- `index_4`: `63.24387741088867`
- `index_5`: `3.774722099304199`
- `index_6`: `63.24387741088867`
- `index_7`: `344.52294921875`
- `index_8`: `63.24387741088867`
- `index_9`: `344.52294921875`
- *(showing 10 of 324 rows using 'uniform' sampling)*


###### Dataset: Transverse

####### Properties

- **Shape:** `(324,)`
- **Data Type:** `float32`
- **Size:** `324` elements

**Data (Key-Value Format):**

- `index_0`: `185.6212158203125`
- `index_1`: `138.95980834960938`
- `index_2`: `138.95980834960938`
- `index_3`: `278.91937255859375`
- `index_4`: `479.72283935546875`
- `index_5`: `185.6212158203125`
- `index_6`: `479.72283935546875`
- `index_7`: `234.235595703125`
- `index_8`: `479.72283935546875`
- `index_9`: `234.235595703125`
- *(showing 10 of 324 rows using 'uniform' sampling)*


##### Group: /Projections/NGFC/NGFC/Edges


###### Dataset: Destination Block Index

####### Properties

- **Shape:** `(78,)`
- **Data Type:** `uint32`
- **Size:** `78` elements
- **Chunks:** `(78,)`

**Data (Key-Value Format):**

- `index_0`: `1207`
- `index_1`: `1680`
- `index_2`: `1708`
- `index_3`: `1734`
- `index_4`: `1774`
- `index_5`: `1810`
- `index_6`: `1852`
- `index_7`: `1874`
- `index_8`: `1911`
- `index_9`: `1956`
- *(showing 10 of 78 rows using 'uniform' sampling)*


###### Dataset: Destination Block Pointer

####### Properties

- **Shape:** `(79,)`
- **Data Type:** `uint64`
- **Size:** `79` elements
- **Chunks:** `(79,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `9`
- `index_2`: `21`
- `index_3`: `33`
- `index_4`: `45`
- `index_5`: `59`
- `index_6`: `70`
- `index_7`: `81`
- `index_8`: `94`
- `index_9`: `108`
- *(showing 10 of 79 rows using 'uniform' sampling)*


###### Dataset: Destination Pointer

####### Properties

- **Shape:** `(108,)`
- **Data Type:** `uint64`
- **Size:** `108` elements
- **Chunks:** `(108,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `32`
- `index_2`: `63`
- `index_3`: `99`
- `index_4`: `150`
- `index_5`: `179`
- `index_6`: `219`
- `index_7`: `252`
- `index_8`: `282`
- `index_9`: `324`
- *(showing 10 of 108 rows using 'uniform' sampling)*


###### Dataset: Source Index

####### Properties

- **Shape:** `(324,)`
- **Data Type:** `uint32`
- **Size:** `324` elements
- **Chunks:** `(324,)`

**Data (Key-Value Format):**

- `index_0`: `1500`
- `index_1`: `1839`
- `index_2`: `1500`
- `index_3`: `1683`
- `index_4`: `1680`
- `index_5`: `1756`
- `index_6`: `1898`
- `index_7`: `1949`
- `index_8`: `2057`
- `index_9`: `2019`
- *(showing 10 of 324 rows using 'uniform' sampling)*
