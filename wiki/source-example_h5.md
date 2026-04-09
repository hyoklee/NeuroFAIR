# HDF5 File Structure: /home/hyoklee/neuroh5/data/example.h5


## Attributes

- **ahdf5-smd-H5Types:** `BEST GUESS: Type registry group containing committed HDF5 compound and enumerated datatypes for neuroh5 population and projection metadata.` (type: `str`)
- **ahdf5-smd-Projections:** `BEST GUESS: Top-level group containing all synaptic projections. Each child group represents one directed projection between two neuron populations.` (type: `str`)
- **ahdf5-smd-root:** `BEST GUESS: neuroh5 HDF5 connectivity file containing a minimal Granule Cell to Granule Cell (GC→GC) projection. Stores synaptic connectivity in Destination Block Sparse format with longitudinal and transverse distance edge attributes.` (type: `str`)


## Group: /H5Types

### Attributes

- **ahdf5-smd-Layer tags:** `BEST GUESS: Committed enumerated HDF5 datatype (uint8) encoding cortical layer tags for neuron classification.` (type: `str`)
- **ahdf5-smd-Population labels:** `BEST GUESS: Committed enumerated HDF5 datatype (uint16) mapping integer codes to neuron population names (e.g. Granule Cell).` (type: `str`)
- **ahdf5-smd-Population projections:** `BEST GUESS: Committed compound HDF5 datatype with fields Source (uint16) and Destination (uint16) identifying a directed projection between two populations.` (type: `str`)
- **ahdf5-smd-Population range:** `BEST GUESS: Committed compound HDF5 datatype with fields Start (uint64), Count (uint32), and Population (uint16) describing the GID range occupied by one population.` (type: `str`)
- **ahdf5-smd-Populations:** `BEST GUESS: Registry dataset (shape 1) listing all neuron populations in this file. Each record gives the starting GID, the number of neurons, and the population label.` (type: `str`)
- **ahdf5-smd-Valid population projections:** `BEST GUESS: Registry dataset (shape 1) enumerating the one valid (Source, Destination) population projection pair in this file.` (type: `str`)


### Dataset: Populations

#### Properties

- **Shape:** `(1,)`
- **Data Type:** `[('Start', '<u8'), ('Count', '<u4'), ('Population', '<u2')]`
- **Size:** `1` elements
- **Chunks:** `(512,)`

**Data (Key-Value Format):**

- `index_0`: `(0, 1000000, 0)`


### Dataset: Valid population projections

#### Properties

- **Shape:** `(1,)`
- **Data Type:** `[('Source', '<u2'), ('Destination', '<u2')]`
- **Size:** `1` elements
- **Chunks:** `(1024,)`

**Data (Key-Value Format):**

- `index_0`: `(0, 0)`


## Group: /Projections

### Attributes

- **ahdf5-smd-GCtoGC:** `BEST GUESS: Granule Cell to Granule Cell recurrent projection. Contains 14 directed synaptic connections stored in Destination Block Sparse format with spatial distance edge attributes.` (type: `str`)


### Group: /Projections/GCtoGC

#### Attributes

- **ahdf5-smd-Attributes:** `BEST GUESS: Edge attribute container for the GC→GC projection. Child groups hold per-edge float datasets aligned with the Source Index array.` (type: `str`)
- **ahdf5-smd-Connectivity:** `BEST GUESS: Destination Block Sparse (DBS) connectivity arrays for the GC→GC projection. Contains Source Index, Destination Block Index, Destination Block Pointer, and Destination Pointer datasets.` (type: `str`)
- **ahdf5-smd-Destination Population:** `BEST GUESS: Scalar attribute or dataset identifying the destination neuron population (Granule Cell) for this projection.` (type: `str`)
- **ahdf5-smd-Source Population:** `BEST GUESS: Scalar attribute or dataset identifying the source neuron population (Granule Cell) for this projection.` (type: `str`)


#### Group: /Projections/GCtoGC/Attributes

##### Attributes

- **ahdf5-smd-Edge:** `BEST GUESS: Group collecting all edge-level attribute datasets for the GC→GC projection.` (type: `str`)


##### Group: /Projections/GCtoGC/Attributes/Edge

###### Attributes

- **ahdf5-smd-Longitudinal Distance:** `BEST GUESS: Per-edge synaptic distance along the longitudinal axis (float32, 14 values, range 60–623 µm). One value per entry in Source Index.` (type: `str`)
- **ahdf5-smd-Transverse Distance:** `BEST GUESS: Per-edge synaptic distance along the transverse axis (float32, 14 values). One value per entry in Source Index.` (type: `str`)


###### Dataset: Longitudinal Distance

####### Properties

- **Shape:** `(14,)`
- **Data Type:** `float32`
- **Size:** `14` elements
- **Compression:** `gzip`
- **Chunks:** `(1024,)`

**Data (Key-Value Format):**

- `index_0`: `318.04034423828125`
- `index_1`: `107.19705963134766`
- `index_2`: `305.0643005371094`
- `index_3`: `140.02093505859375`
- `index_4`: `203.9142608642578`
- `index_5`: `75.158935546875`
- `index_6`: `224.36227416992188`
- `index_7`: `236.71099853515625`
- `index_8`: `65.5165786743164`
- `index_9`: `60.09276580810547`
- *(showing 10 of 14 rows using 'uniform' sampling)*


###### Dataset: Transverse Distance

####### Properties

- **Shape:** `(14,)`
- **Data Type:** `float32`
- **Size:** `14` elements
- **Compression:** `gzip`
- **Chunks:** `(1024,)`

**Data (Key-Value Format):**

- `index_0`: `271.67901611328125`
- `index_1`: `65.64421844482422`
- `index_2`: `256.7221374511719`
- `index_3`: `104.88873291015625`
- `index_4`: `234.98841857910156`
- `index_5`: `133.4642791748047`
- `index_6`: `201.74606323242188`
- `index_7`: `114.34191131591797`
- `index_8`: `62.18435287475586`
- `index_9`: `16.575515747070312`
- *(showing 10 of 14 rows using 'uniform' sampling)*


#### Group: /Projections/GCtoGC/Connectivity

##### Attributes

- **ahdf5-smd-Destination Block Index:** `BEST GUESS: First destination GID of each contiguous destination block in the DBS representation (uint32). Used with Destination Block Pointer to locate per-destination slices.` (type: `str`)
- **ahdf5-smd-Destination Block Pointer:** `BEST GUESS: Offsets (uint32, shape 2) into the Destination Pointer array marking the start of each destination block. The difference between consecutive values gives the number of destinations in that block.` (type: `str`)
- **ahdf5-smd-Destination Pointer:** `BEST GUESS: Offsets (uint32) into the Source Index array for each destination neuron. The slice Source Index[dst_ptr[i]:dst_ptr[i+1]] gives all sources connected to destination i.` (type: `str`)
- **ahdf5-smd-Source Index:** `BEST GUESS: Flat array of source GIDs for all 14 edges in the GC→GC projection (uint32, range 10–19). Length equals the total number of synaptic connections.` (type: `str`)


##### Dataset: Destination Block Index

###### Properties

- **Shape:** `(1,)`
- **Data Type:** `uint32`
- **Size:** `1` elements
- **Compression:** `gzip`
- **Chunks:** `(1024,)`

**Data (Key-Value Format):**

- `index_0`: `12`


##### Dataset: Destination Block Pointer

###### Properties

- **Shape:** `(2,)`
- **Data Type:** `uint32`
- **Size:** `2` elements
- **Compression:** `gzip`
- **Chunks:** `(1024,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `9`


##### Dataset: Destination Pointer

###### Properties

- **Shape:** `(9,)`
- **Data Type:** `uint64`
- **Size:** `9` elements
- **Compression:** `gzip`
- **Chunks:** `(1024,)`

**Data (Key-Value Format):**

- `index_0`: `0`
- `index_1`: `1`
- `index_2`: `2`
- `index_3`: `3`
- `index_4`: `4`
- `index_5`: `7`
- `index_6`: `9`
- `index_7`: `13`
- `index_8`: `14`


##### Dataset: Source Index

###### Properties

- **Shape:** `(14,)`
- **Data Type:** `uint32`
- **Size:** `14` elements
- **Compression:** `gzip`
- **Chunks:** `(1024,)`

**Data (Key-Value Format):**

- `index_0`: `10`
- `index_1`: `12`
- `index_2`: `13`
- `index_3`: `11`
- `index_4`: `14`
- `index_5`: `10`
- `index_6`: `13`
- `index_7`: `15`
- `index_8`: `16`
- `index_9`: `16`
- *(showing 10 of 14 rows using 'uniform' sampling)*


#### Dataset: Destination Population

##### Properties

- **Shape:** `(1,)`
- **Data Type:** `uint32`
- **Size:** `1` elements
- **Chunks:** `(1024,)`

**Data (Key-Value Format):**

- `index_0`: `0`


#### Dataset: Source Population

##### Properties

- **Shape:** `(1,)`
- **Data Type:** `uint32`
- **Size:** `1` elements
- **Chunks:** `(1024,)`

**Data (Key-Value Format):**

- `index_0`: `0`
