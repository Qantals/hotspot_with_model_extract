# HotSpot

## detailed_3D
1. function `alloc_grid_model()` in `populate_layers_grid()` in `append_package_layers()`
    - set spreader b2gmap is new (different from top inner layer)
    - original 3D attach spreader to top inner layer
    - both original 3D and detailed_3D attach heat sinker and other layers (`model_secondary`) to heat spreader layer b2gmap
1. function `populate_R_model_grid()`
    - set spreader b2gmap with new one, set all `b2gmap[i][j]->hasRes, hasCap, LOCK = FALSE`
1. function `set_bgmap()`:
    - for specific layer set g2bmap (array num = blocks, with grid coordinate of that block) and b2gmap (array num = grids)
    - call `reset_b2gmap()` free each b2gmap and set `NULL`
    - for each unit
        - set g2bmap
        - iterate this unit occupy grids, set b2gmap call `new_blist()` and `blist_append()`
            - original 3D: set idx, occupancy
            - detailed_3D
                - set `hasRes==TRUE` and `hasCap` both original 3D or detailed_3D
                - `hasCap==TRUE` depends on lcf and detailed_3D option.
2. function `new_blist()`
    - `b2gmap->idx, occupancy`: both original 3D and detailed_3D have, both head pointer and others have. Detailed_3D do not change.
    - `b2gmap->LOCK, rx, ry, rz, capacitance` can be set only if using **detailed_3D and head pointer**.
        - `b2gmap->LOCK==TRUE` dependent of occupancy, set if occupancy > OCCUPANCY_THRESHOLD.
        - `b2gmap->rx, ry, rz, capacitance` can be layer default config (not specified in LCF) or unit specific.
            - `rx, ry, rz` dependent of occupancy, not occupancy > OCCUPANCY_THRESHOLD just to be parallel adding
            - `capacitance` independent of occupancy
3. function `blist_append()`
    - iterate from head pointer, change head pointer if not detailed_3D and `LOCK==FALSE`:
        - if `occupancy > OCCUPANCY_THRESHOLD`: set `LOCK=TRUE`, change `rx, ry, rz, capacitance`
        - else add parallel resistance `rx, ry, rz` but not `capacitance`
    - append at tail, call `new_blist()`
    - side comment: `detailed_3D` with poor coding: put new code after original `head` pointer NULL check!

## transient
1. function `main()`
    - vector `temp, power, steady_temp, overall_power` with size = `model->total_n_units` == sum of all flp units in all layers and additional extra nodes.
        - `overall_power`: get average power across all time steps for each unit in flp across all power dissipating layers
    - `-init` only read `.steady` file, rather than `.grid.steady` file
    - if not `-init` file, use config ambient temperature
2. function `steady_state_temp_grid()` get steady temperature
    - use `overall_power`, dump to `steady_temp` variable in `main()`
    - `p` variable with size == all grids: type `grid_model_vector_t`, the same type as `grid_model->last_steady, last_trans`
    - call `set_internal_power_grid()` set package node power to 0 for `overall_power` in `main()`
    - map power: block to grid call `xlate_vector_b2g()`
        - power: iterate all grids, call `blist_avg()`
        - search grid `b2gmap[l][i][j]->occupancy, idx` iterate all linked list
        - get `overall_power[base+idx]` where `base` is layer offset
        - compute `flp->unit[idx]->width, height` to get power density
        - times `occupancy` for power contribution to this grid
        - outside `blist_avg()` compute grid area transform power density to power
    - compute AT=P: use `p`, store in `grid_model->last_steady`
        - build A matrix: call `build_transient_grid_matrix()` call `find_res()` call `find_res_3D()`
        - build P vector: use `p` call `build_transient_power_vector()`
    - call `xlate_temp_g2b()` translate from grid to block. OK to modification because it iterates flp units and g2bmap, as long as `model->last_steady` is OK.
3. function `compute_temp()` call `compute_temp_grid()` get transient temperature
    <!-- - for my purpose of **modification**, can change behavior in function `main()`, do not use function `compute_temp(temp=temp)` (not change `compute_temp(temp=NULL)`), use my create one. -->
    - first call `main()` pass `temp` variable in `main()` filled with ambient temperature
        <!-- - for my purpose of **modification**, can change behavior in function `compute_temp_grid()`, do not use function `xlate_vector_b2g()`, use my create one. -->
        - call `xlate_vector_b2g()` store in `model->last_trans`
            - for my purpose of **modification**, can add `model->config->init_temp` parameter to funciton `blist_avg()` set default val as ambient temperature,
                and add occupancy sum count to fix value. Reason: only this point call `xlate_vector_b2g(type=V_TEMP)` so just modify this part
        - store block temperature `temp` in `model->last_temp`
    - other calls pass in NULL because this function use `model->last_trans`
    - first steps same as `steady_state_temp_grid()`: but use `power` instead of `overall_power`, create `p` set package 0, map power
    - use static variable `first_call` record function call times, only compute matrix G, C once, store in `model->G, C`
        - G matrix: call `build_transient_grid_matrix()`, same as steady
        - C matrix: call `build_diagonal_matrix()` call `find_cap_3D()`
        - T vector: from `model->last_trans`
        - P vector: call `build_transient_power_vector(p)`, same as steady
    - call `xlate_temp_g2b()` translate from grid: `model->last_trans` to block: `model->last_temp`, same as steady
