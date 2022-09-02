### Background

This code uses network navigation to select upstream and downstream
segments within a 5 km range of each site and generates distances for
each stressor within 5km of each site as follows: 1. Downstream
crossings: a record (row) for every downstream crossing with columns of
BCG site and values would be distance downstream 2. Upstream crossings:
same as 1, but upstream crossings with distance upstream 3. Small Lakes
Upstream: record (row) for every small upstream lake with columns for
each BCG site and then value of distance upstream 4. Large Downstream
Lakes: record (row) of every large downstream lake with columns for each
BCG site and value of distance downstream

### Setup

    library(readr)
    library(nhdplusTools)
    library(sf)
    library(dplyr)
    library(tigris)
    library(mapview)
    library(leaflet)
    library(ggplot2)
    library(rmapshaper)
    library(reticulate)
    sessionInfo()

    ## R version 4.0.2 (2020-06-22)
    ## Platform: x86_64-w64-mingw32/x64 (64-bit)
    ## Running under: Windows 10 x64 (build 19042)
    ## 
    ## Matrix products: default
    ## 
    ## locale:
    ## [1] LC_COLLATE=English_United States.1252 
    ## [2] LC_CTYPE=English_United States.1252   
    ## [3] LC_MONETARY=English_United States.1252
    ## [4] LC_NUMERIC=C                          
    ## [5] LC_TIME=English_United States.1252    
    ## 
    ## attached base packages:
    ## [1] stats     graphics  grDevices utils     datasets  methods   base     
    ## 
    ## other attached packages:
    ##  [1] reticulate_1.20    rmapshaper_0.4.5   ggplot2_3.3.5      leaflet_2.0.4.1   
    ##  [5] mapview_2.10.0     tigris_1.5         dplyr_1.0.7        sf_1.0-2          
    ##  [9] nhdplusTools_0.4.3 readr_2.0.1       
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] httr_1.4.2         tidyr_1.1.3        geojsonlint_0.4.0  jsonlite_1.7.2    
    ##  [5] assertthat_0.2.1   sp_1.4-5           stats4_4.0.2       yaml_2.2.1        
    ##  [9] pillar_1.6.2       lattice_0.20-41    glue_1.4.2         uuid_0.1-4        
    ## [13] digest_0.6.27      colorspace_2.0-0   Matrix_1.2-18      htmltools_0.5.2   
    ## [17] pkgconfig_2.0.3    httpcode_0.3.0     raster_3.4-13      purrr_0.3.4       
    ## [21] scales_1.1.1       webshot_0.5.2      RANN_2.6.1         satellite_1.0.2   
    ## [25] tzdb_0.1.2         tibble_3.1.3       generics_0.1.0     ellipsis_0.3.2    
    ## [29] withr_2.4.1        fst_0.9.4          magrittr_2.0.1     crayon_1.4.1      
    ## [33] maptools_1.1-1     evaluate_0.14      fansi_0.4.2        xml2_1.3.2        
    ## [37] foreign_0.8-80     class_7.3-17       tools_4.0.2        hms_1.1.0         
    ## [41] lifecycle_1.0.0    stringr_1.4.0      V8_3.4.2           munsell_0.5.0     
    ## [45] zip_2.1.1          compiler_4.0.2     e1071_1.7-4        rlang_0.4.11      
    ## [49] classInt_0.4-3     units_0.7-0        grid_4.0.2         rappdirs_0.3.3    
    ## [53] htmlwidgets_1.5.3  crosstalk_1.1.1    igraph_1.2.6       leafem_0.1.3      
    ## [57] base64enc_0.1-3    rmarkdown_2.10     gtable_0.3.0       codetools_0.2-18  
    ## [61] jsonvalidate_1.3.2 DBI_1.1.1          curl_4.3.2         R6_2.5.0          
    ## [65] knitr_1.33         rgdal_1.5-23       fastmap_1.1.0      utf8_1.1.4        
    ## [69] KernSmooth_2.23-17 stringi_1.7.3      crul_1.1.0         parallel_4.0.2    
    ## [73] Rcpp_1.0.7         vctrs_0.3.8        png_0.1-7          tidyselect_1.1.1  
    ## [77] xfun_0.25

### Load data

The data used is from [King County Freshwater Benthic Macroinvertebrate
Sampling and Analysis Plan](https://pugetsoundstreambenthos.org/). The
data citation is: King County. 2020. King County Freshwater Benthic
Macroinvertebrate Sampling and Analysis Plan. Prepared by Kate Macneale,
Liora Llewellyn, and Beth Sosik. Water and Land Resources Division.
Seattle, Washington. <https://pugetsoundstreambenthos.org/>

    sites <- read_csv('Multiscale Framework King County and Puget Lowland data.csv')
    sites <- sites %>% 
      dplyr::select(SITE_ID = Location_ID, Latitude, Longitude, COMID)
    sites <- sites %>%
      st_as_sf(coords = c("Longitude", "Latitude"), crs = 4269, remove = FALSE)

### Create Road-Stream Crossings

Download US Census Bureau Tiger Line files representing roads and
calculate crossings with NHDPlusHR stream lines

    Sys.getenv('TIGRIS_CACHE_DIR')
    options(tigris_use_cache=TRUE)
    options(tigris_refresh=TRUE)
    counties <- tigris::counties(state='Washington')
    sites <- st_join(sites, counties[,c('NAME')])
    county_list <- levels(as.factor(sites$NAME))
    for (i in 1:length(county_list)){
      if (i==1) roads <- tigris::roads("WA",county_list[i]) else{
        temp <- roads <- tigris::roads("WA",county_list[i])
        roads <- rbind(roads, temp)
      }
    }

    flowlines_1708 <- read_sf(dsn='NHDPLUS_H_1708_HU4_GDB.gdb',layer='NHDFlowline')
    flowlines_1710 <- read_sf(dsn='NHDPLUS_H_1710_HU4_GDB.gdb',layer='NHDFlowline')
    flowlines_1711 <- read_sf(dsn='NHDPLUS_H_1711_HU4_GDB.gdb',layer='NHDFlowline')

    flowlines <- rbind(flowlines_1708, flowlines_1710, flowlines_1711)

    flowlines <- st_zm(flowlines, drop=TRUE, what="ZM")
    road_strms <- st_intersection(flowlines, roads)
    rm(flowlines_1708, flowlines_1710, flowlines_1711)
    flowlines <- st_transform(flowlines, st_crs(sites))
    indexes <- get_flowline_index(flowlines,                             sf::st_geometry(sites),
                                  search_radius = 300,
                                  max_matches = 1)
    sites$id <- 1:nrow(sites)
    sites <- left_join(sites,indexes, by = "id")

## NABD dams

Apply weights to dams and reference to NHDPlusHR in Puget Sound

    dams <- read_sf('NABD.shp')
    sites <- st_transform(sites, st_crs(dams))
    dams <- dams %>% 
      dplyr::filter(State=='WA')
    dams <- st_crop(dams, st_bbox(sites))
    # generate weights, drop unneeded fields
    dams <- dams %>% 
      mutate(NID_height=NID_height*0.30488) %>%
      mutate(down_weight = case_when(
        NID_height >= 15 ~ 0,
        NID_height < 15 ~ .5,
         TRUE ~ 1)) %>%
      mutate(up_weight = case_when(
        NID_height >= 15 ~ .25,
        NID_height < 15 ~ .8,
         TRUE ~ 1)) %>%
      dplyr::select(NIDID,Longitude,Latitude,Dam_name,Dam_type,NID_height,Dam_length,NrmStorM3,NIDStorM3,down_weight, up_weight)

    flowlines <- st_transform(flowlines, st_crs(dams))

### Linear referencing

Use linear referencing to reference sites and dams to NHDPlusHR
flowlines in Puget Sound

    import arcpy
    # Execute LocateFeaturesAlongRoutes
    pts = 'Puget_NABD.shp'
    nhd1 = 'L:/Priv/CORFiles/Geospatial_Library_Projects/IWI/BCGs/Puget 2020/Marc/NHDPLUS_H_1708_HU4_GDB.gdb/NHDFlowline'
    nhd2 = 'L:/Priv/CORFiles/Geospatial_Library_Projects/IWI/BCGs/Puget 2020/Marc/NHDPLUS_H_1710_HU4_GDB.gdb/NHDFlowline'
    nhd3 = 'L:/Priv/CORFiles/Geospatial_Library_Projects/IWI/BCGs/Puget 2020/Marc/NHDPLUS_H_1711_HU4_GDB.gdb/NHDFlowline'

    props = "RID POINT MEAS"
    arcpy.LocateFeaturesAlongRoutes_lr(pts, nhd1, "REACHCODE", "1000 Meters", "Puget_NABD_LR1.dbf", props, "FIRST", "DISTANCE", "ZERO", "FIELDS", "M_DIRECTON")
    arcpy.LocateFeaturesAlongRoutes_lr(pts, nhd2, "REACHCODE", "1000 Meters", "Puget_NABD_LR2.dbf", props, "FIRST", "DISTANCE", "ZERO", "FIELDS", "M_DIRECTON")
    arcpy.LocateFeaturesAlongRoutes_lr(pts, nhd3, "REACHCODE", "1000 Meters", "Puget_NABD_LR3.dbf", props, "FIRST", "DISTANCE", "ZERO", "FIELDS", "M_DIRECTON")

    lr1 <- read_sf('Puget_NABD_LR1.dbf')
    lr2 <- read_sf('Puget_NABD_LR2.dbf')
    lr3 <- read_sf('Puget_NABD_LR3.dbf')
    lr <- rbind(lr1, lr2, lr3)
    dams$RID <- lr$RID[match(dams$NIDID, lr$NIDID)]
    dams$MEAS <- lr$MEAS[match(dams$NIDID, lr$NIDID)]
    dams$Distance <- lr$Distance[match(dams$NIDID, lr$NIDID)]

    dams <- st_join(dams, flowlines[,c('NHDPlusID')], join=st_nearest_feature)
    st_write(dams, 'Puget_NABD.shp')

### NHDPlusHR Pathlength and DownLevelPath

Derive Pathlength and DownLevelPath from NHDPlusHR

    flowlines$PathLength <- vaa$PathLength[match(flowlines$NHDPlusID, vaa$NHDPlusID)]
    flowlines$HydroSeq <- vaa$HydroSeq[match(flowlines$NHDPlusID, vaa$NHDPlusID)]
    flowlines$LevelPathI <- vaa$LevelPathI[match(flowlines$NHDPlusID, vaa$NHDPlusID)]
    flowlines$DnHydroSeq <- vaa$DnHydroSeq[match(flowlines$NHDPlusID, vaa$NHDPlusID)]
    flowlines$DnLevelPat <- vaa$DnLevelPat[match(flowlines$NHDPlusID, vaa$NHDPlusID)]

### NHDPlusHR Waterbodies and Catchments

    st_line_midpoints <- function(sf_lines = NULL) {
      g <- st_geometry(sf_lines)
      g_mids <- lapply(g, function(x) {
        coords <- as.matrix(x)

        # this is just a copypaste of View(maptools:::getMidpoints):
        get_mids <- function (coords) {
          dist <- sqrt((diff(coords[, 1])^2 + (diff(coords[, 2]))^2))
          dist_mid <- sum(dist)/2
          dist_cum <- c(0, cumsum(dist))
          end_index <- which(dist_cum > dist_mid)[1]
          start_index <- end_index - 1
          start <- coords[start_index, ]
          end <- coords[end_index, ]
          dist_remaining <- dist_mid - dist_cum[start_index]
          mid <- start + (end - start) * (dist_remaining/dist[start_index])
          return(mid)
        }

        mids <- st_point(get_mids(coords))
        })

      out <- st_sfc(g_mids, crs = st_crs(sf_lines))
      out <- st_sf(out)
    }

    wb1 <- read_sf(dsn='NHDPLUS_H_1708_HU4_GDB.gdb',layer='NHDWaterbody')
    wb2 <- read_sf(dsn='NHDPLUS_H_1710_HU4_GDB.gdb',layer='NHDWaterbody')
    wb3 <- read_sf(dsn='NHDPLUS_H_1711_HU4_GDB.gdb',layer='NHDWaterbody')
    wb <- rbind(wb1, wb2, wb3)
    rm(wb1, wb2, wb3)

    fcode <- read_sf(dsn='NHDPLUS_H_1711_HU4_GDB.gdb',layer='NHDFcode')
    wb$Desc <- fcode$Description[match(wb$FCode, fcode$FCode)]
    levels(as.factor(wb$Desc))
    wb <- wb %>% 
      dplyr::filter(!Desc %in% c('Estuary','Ice Mass')) %>% 
      dplyr::rename(WBArea_Permanent_Identifier=Permanent_Identifier)
    wb$Desc <- as.factor(wb$Desc)
    levels(wb$Desc) <- c("Lake/Pond", "Lake/Pond Intermittent", "Lake/Pond Intermittent","Lake/Pond Intermittent","Lake/Pond Perennial","Lake/Pond Perennial","Lake/Pond Perennial","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Reservoir","Swamp/Marsh","Swamp/Marsh Intermittent","Swamp/Marsh Perennial")
    wb$Desc <- as.character(wb$Desc)

    wb_swamps <- flowlines %>% 
      st_line_midpoints()
    flowline_atts <- flowlines %>% 
      dplyr::select(Permanent_Identifier,NHDPlusID,HydroSeq) %>%
      st_drop_geometry()
    wb_swamps$Permanent_Identifier <- flowline_atts$Permanent_Identifier
    wb_swamps$NHDPlusID <- flowline_atts$NHDPlusID
    wb_swamps$HydroSeq <- flowline_atts$HydroSeq
    # swamp / marshes don't have WBArea_Permanent_Identifier populated in NHDPlusHR
    wb <- st_transform(wb, st_crs(flowlines))
    wb_swamps <-st_join(wb_swamps, wb[,c('WBArea_Permanent_Identifier','GNIS_ID','GNIS_Name','AreaSqKm','Desc')])

    # wb_swamps <- left_join(wb_swamps, st_drop_geometry(wb[,c('WBArea_Permanent_Identifier','AreaSqKm','Desc')]))

    # Get Startflag to identify headwaters
    wb_swamps$StartFlag <- vaa$StartFlag[match(wb_swamps$NHDPlusID, vaa$NHDPlusID)]

    wb_swamps <- wb_swamps %>%
      dplyr::filter(!is.na(WBArea_Permanent_Identifier))

    wb_swamps$HydroSeq[is.na(wb_swamps$HydroSeq)] <- 0
    wb_swamps$StartFlag[is.na(wb_swamps$StartFlag)] <- 0

    canals <- flowlines %>% 
      dplyr::filter(FType==336) %>% 
      dplyr::select(Type=FType)

    cat1 <- read_sf(dsn='NHDPLUS_H_1708_HU4_GDB.gdb',layer='NHDPlusCatchment')
    cat2 <- read_sf(dsn='NHDPLUS_H_1710_HU4_GDB.gdb',layer='NHDPlusCatchment')
    cat3 <- read_sf(dsn='NHDPLUS_H_1711_HU4_GDB.gdb',layer='NHDPlusCatchment')
    cats <- rbind(cat1, cat2, cat3)
    rm(cat1, cat2, cat3)
    canals <- st_transform(canals, st_crs(cats))
    cats <- st_join(cats, canals)
    canals <- cats %>% 
      dplyr::filter(!is.na(Type) & !duplicated(NHDPlusID))

### Associate NHDPlusV21 MMI with NHDPlusHR flowlines

    mmi <- read_csv('NRSA_PredictedBioCondition_Region_allstreams17.csv')
    # get downstream points of streams
    flowlines_simple <- st_cast(flowlines, "LINESTRING")
    stream_points <- st_line_sample(flowlines_simple, sample = 1)
    stream_points <- st_sf(stream_points)
    stream_points$NHDPlusID <- flowlines$NHDPlusID[match(row.names(stream_points),row.names(flowlines))]
    cats <- read_sf('Catchment.shp')
    cats <- st_transform(cats, st_crs(stream_points))
    NHD_catCOM <- st_join(stream_points, cats)
    NHD_catCOM <- NHD_catCOM %>% 
      dplyr::select(NHDPlusID,COMID=FEATUREID)
    NHD_catCOM$MMI <- mmi$prG_BMMI[match(NHD_catCOM$COMID, mmi$COMID)]
    flowlines$MMI <- NHD_catCOM$MMI[match(flowlines$NHDPlusID, NHD_catCOM$NHDPlusID)]

### Downstream Dispersal Calculation

Gather Upstream segments within 5km of sites to calculate downstream
scores

    sites <- sites %>% 
      dplyr::filter(!duplicated(SITE_ID))
    sites$ddi <- 0
    sites$upstream_count <- 0
    sites$upstream_NA <- 0
    sites$upstream_small_dams <- 0
    sites$upstream_large_dams <- 0
    sites$upstream_canals <- 0
    sites$upstream_lakes <- 0
    sites$upstream_swamps <- 0
    sites$upstream_crossings <- 0
    sites$upstream_flowlength <- 0
    sites$upstream_mean_MMI <- 0

    sites_buf <- st_buffer(sites, dist=5000)

    for (k in 1:nrow(sites)){
      print (paste0('working on ', as.character(st_drop_geometry(sites[k, c('SITE_ID')]))))
      
      temp_com <- as.double(st_drop_geometry(sites[k,c('COMID')]))
      temp_reach <- as.double(st_drop_geometry(sites[k,c('REACHCODE')]))
      temp_reach_meas <- as.double(st_drop_geometry(sites[k,c('REACH_meas')]))
      
      Upstream <- get_UT(flowlines, temp_com, distance = 10)
      if (Upstream==55000800000008){
        Upstream2 <- get_UT(flowlines,  55000800051691, distance = 10)
        Upstream = c(Upstream, Upstream2)
      }

      Up_streams <- flowlines %>% 
        dplyr::filter(NHDPlusID %in% Upstream)
      Up_streams <- st_intersection(Up_streams, sites_buf[k,])
      
      Up_dams <- dams %>% 
        dplyr::filter(NHDPlusID %in% Up_streams$NHDPlusID & !(RID == temp_reach & MEAS < temp_reach_meas))
      Up_dams <- Up_dams[sites_buf[k,],]
      
      Up_crossings <- road_strms %>% 
        dplyr::filter(NHDPlusID %in% Up_streams$NHDPlusID & !(NHDPlusID == temp_com & REACH_meas < temp_reach_meas))
      Up_crossings <- Up_crossings[sites_buf[k,],]
      
      Up_canals <- canals %>% 
        dplyr::filter(NHDPlusID %in% Up_streams$NHDPlusID)
      Up_canals <- st_transform(Up_canals, st_crs(sites_buf))
      Up_canals <- Up_canals[sites_buf[k,],]
      
      Up_wb_swamp <- wb_swamps %>% 
        dplyr::filter(NHDPlusID %in% Up_streams$NHDPlusID) %>%
      dplyr::group_by(WBArea_Permanent_Identifier) %>% 
        dplyr::filter(any(StartFlag!=1)) %>%
        dplyr::filter(HydroSeq == min(HydroSeq))
      
      #dist_decay
      Up_streams$dist <- Up_streams$PathLength - min(Up_streams$PathLength)

      Up_streams <- Up_streams %>% 
        mutate(dist_decay = case_when(
          dist < 1 ~ 1,
          dist >= 1 ~ (1/dist^2)))
      
      Up_streams$ddi <- 1
      if (nrow(Up_dams) >0){
        for (i in 1:nrow(Up_dams)){
          temp_weight <- as.double(st_drop_geometry(Up_dams[i, c('down_weight')]))
          up_temp <- get_UT(Up_streams, as.double(st_drop_geometry(Up_dams[i, c('NHDPlusID')])), distance = 5)
          Up_streams$ddi[Up_streams$NHDPlusID %in% up_temp] <- Up_streams$ddi[Up_streams$NHDPlusID %in% up_temp] * temp_weight 
        }
      }

      if (nrow(Up_crossings) > 0){  
        for (i in 1:nrow(Up_crossings)){
          temp_weight <- .6
          up_temp <- get_UT(Up_streams, as.double(st_drop_geometry(Up_crossings[i, c('NHDPlusID')])), distance = 5)
          print (i)
          print (up_temp)
          Up_streams$ddi[Up_streams$NHDPlusID %in% up_temp] <- Up_streams$ddi[Up_streams$NHDPlusID %in% up_temp] * temp_weight
        }
      }

      if (nrow(Up_canals) >0){
        for (i in 1:nrow(Up_canals)){
          temp_weight <- .4
          up_temp <- get_UT(Up_streams, as.double(st_drop_geometry(Up_canals[i, c('NHDPlusID')])), distance = 5)
          print (i)
          print (up_temp)
          Up_streams$ddi[Up_streams$NHDPlusID %in% up_temp] <- Up_streams$ddi[Up_streams$NHDPlusID %in% up_temp] * temp_weight
        }
      }

      if (nrow(Up_wb_swamp) >0){
        for (i in 1:nrow(Up_wb_swamp)){
          if (st_drop_geometry(Up_wb_swamp[i,c('Desc')]) %in% c('Lake/Pond Perennial','Lake/Pond Intermittent')) temp_weight <- .8
          if (st_drop_geometry(Up_wb_swamp[i,c('Desc')]) %in% c("Swamp/Marsh", "Swamp/Marsh Perennial","Swamp/Marsh Intermittent")) temp_weight <- .6
          up_temp <- get_UT(Up_streams, as.double(st_drop_geometry(Up_wb_swamp[i, c('NHDPlusID')])), distance = 5)
          print (i)
          print (up_temp)
          Up_streams$ddi[Up_streams$NHDPlusID %in% up_temp] <- Up_streams$ddi[Up_streams$NHDPlusID %in% up_temp] * temp_weight
        }
      }

      # Incorporate MMI, stream Length, and dist decay
      Up_streams$MMI_dummy <- Up_streams$MMI
      Up_streams$MMI_dummy[is.na(Up_streams$MMI_dummy)] <- 1
      Up_streams$ddi <- Up_streams$ddi * Up_streams$MMI_dummy * Up_streams$dist_decay * Up_streams$LengthKM
      
      sites[k,c('ddi')] <- sum(Up_streams$ddi,na.rm = FALSE)
      sites[k,c('upstream_count')] <- nrow(Up_streams)
      sites[k,c('upstream_NA')] <- nrow(Up_streams[is.na(Up_streams$MMI),])
      sites[k,c('upstream_large_dams')] <- nrow(Up_dams[Up_dams$down_weight==0,])
      sites[k,c('upstream_small_dams')] <- nrow(Up_dams[Up_dams$down_weight==0.5,])
      sites[k,c('upstream_canals')] <- nrow(Up_canals)
      sites[k,c('upstream_lakes')] <- nrow(Up_wb_swamp[Up_wb_swamp$Desc %in% c('Lake/Pond Perennial','Lake/Pond Intermittent'),])
      sites[k,c('upstream_swamps')] <- nrow(Up_wb_swamp[Up_wb_swamp$Desc %in% c("Swamp/Marsh","Swamp/Marsh Perennial","Swamp/Marsh Intermittent"),])
      sites[k,c('upstream_crossings')] <- nrow(Up_crossings)
      sites[k,c('upstream_flowlength')] <- as.numeric(units::set_units(sum(st_length(Up_streams)), km))
      sites[k,c('upstream_mean_MMI')] <- mean(Up_streams$MMI,na.rm = T)
    }  

### Upstream Dispersal calculation

Gather Downstream segments within 5km of sites to calculate upstream
scores

    sites$udi <- 0
    sites$downstream_count <- 0
    sites$downstream_NA <- 0
    sites$downstream_small_dams <- 0
    sites$downstream_large_dams <- 0
    sites$downstream_canals <- 0
    sites$downstream_large_lakes <- 0
    sites$downstream_small_lakes <- 0
    sites$downstream_swamps <- 0
    sites$downstream_crossings <- 0
    sites$downstream_flowlength <- 0
    sites$downstream_mean_MMI <- 0

    for (k in 1:nrow(sites)){
      print (paste0('working on ', as.character(st_drop_geometry(sites[k, c('SITE_ID')]))))
      temp_com <- as.double(st_drop_geometry(sites[k,c('COMID')]))
      temp_reach <- as.double(st_drop_geometry(sites[k,c('REACHCODE')]))
      temp_reach_meas <- as.double(st_drop_geometry(sites[k,c('REACH_meas')]))
      
      Downstream <- get_DM(flowlines, temp_com, distance = 10)
      
      Down_streams <- flowlines %>% 
        dplyr::filter(NHDPlusID %in% Downstream)
      Down_streams <- st_intersection(Down_streams, sites_buf[k,])
      #dist_decay
      Down_streams$dist <- abs(Down_streams$PathLength - max(Down_streams$PathLength))
      
      Down_streams <- Down_streams %>% 
        mutate(dist_decay = case_when(
          dist < 1 ~ 1,
          dist >= 1 ~ (1/dist^2)))
      
      Down_dams <- dams %>% 
        dplyr::filter(NHDPlusID %in% Down_streams$NHDPlusID & !(RID == temp_reach & MEAS > temp_reach_meas))
      Down_dams <- Down_dams[sites_buf[k,],]
      
      Down_crossings <- road_strms %>% 
        dplyr::filter(NHDPlusID %in% Down_streams$NHDPlusID & !(NHDPlusID == temp_com & REACH_meas > temp_reach_meas))
      Down_crossings <- Down_crossings[sites_buf[k,],]
      
      Down_canals <- st_transform(Down_canals, st_crs(sites_buf))
      Down_canals <- canals %>% 
        dplyr::filter(NHDPlusID %in% Down_streams$NHDPlusID)
      Down_canals <- Down_canals[sites_buf[k,],]
      
      Down_wb_swamp <- wb_swamps %>%
        dplyr::filter(NHDPlusID %in% Down_streams$NHDPlusID) %>%
      group_by(WBArea_Permanent_Identifier) %>%
        dplyr::filter(HydroSeq == max(HydroSeq))
      
      Down_streams$udi <- 1
      if (nrow(Down_dams) >0){
        for (i in 1:nrow(Down_dams)){
          temp_weight <- as.double(st_drop_geometry(Down_dams[i, c('up_weight')]))
          down_temp <- get_DM(Down_streams, as.double(st_drop_geometry(Down_dams[i, c('NHDPlusID')])), distance = 5)
          Down_streams$udi[Down_streams$NHDPlusID %in% down_temp] <- Down_streams$udi[Down_streams$NHDPlusID %in% down_temp] * temp_weight 
        }
      }
      
      if (nrow(Down_crossings) >0){  
        for (i in 1:nrow(Down_crossings)){
          temp_weight <- .4
          down_temp <- get_DM(Down_streams, as.double(st_drop_geometry(Down_crossings[i, c('NHDPlusID')])), distance = 5)
          print (i)
          print (down_temp)
          Down_streams$udi[Down_streams$NHDPlusID %in% down_temp] <- Down_streams$udi[Down_streams$NHDPlusID %in% down_temp] * temp_weight
        }
      }
      
      if (nrow(Down_canals) >0){
        for (i in 1:nrow(Down_canals)){
          temp_weight <- .9
          down_temp <- get_DM(Down_streams, as.double(st_drop_geometry(Down_canals[i, c('NHDPlusID')])), distance = 5)
          print (i)
          print (down_temp)
          Down_streams$udi[Down_streams$NHDPlusID %in% down_temp] <- Down_streams$udi[Down_streams$NHDPlusID %in% down_temp] * temp_weight
        }
      }
      
      if (nrow(Down_wb_swamp) >0){
        for (i in 1:nrow(Down_wb_swamp)){
          if (st_drop_geometry(Down_wb_swamp[i,c('Desc')]) %in% c('Lake/Pond Perennial','Lake/Pond Intermittent') & st_drop_geometry(Down_wb_swamp[i,c('AreaSqKm')]) >- 19) temp_weight <- .1
          if (st_drop_geometry(Down_wb_swamp[i,c('Desc')]) %in% c('Lake/Pond Perennial','Lake/Pond Intermittent') & st_drop_geometry(Down_wb_swamp[i,c('AreaSqKm')]) < 2) temp_weight <- .9    
          if (st_drop_geometry(Down_wb_swamp[i,c('Desc')]) %in% c("Swamp/Marsh","Swamp/Marsh Perennial","Swamp/Marsh Intermittent")) temp_weight <- .75
          down_temp <- get_DM(Down_streams, as.double(st_drop_geometry(Down_wb_swamp[i, c('NHDPlusID')])), distance = 5)
          print (i)
          print (down_temp)
          Down_streams$udi[Down_streams$NHDPlusID %in% down_temp] <- Down_streams$udi[Down_streams$NHDPlusID %in% down_temp] * temp_weight
        }
      }
      
      # Incorporate MMI, stream Length, and dist decay
      
      Down_streams$MMI_dummy <- Down_streams$MMI
      Down_streams$MMI_dummy[is.na(Down_streams$MMI_dummy)] <- 1
      Down_streams$udi <- Down_streams$udi * Down_streams$MMI_dummy * Down_streams$dist_decay * Down_streams$LengthKM
      sites[k,c('udi')] <- sum(Down_streams$udi,na.rm = FALSE)
      
      sites[k,c('downstream_count')] <- nrow(Down_streams)
      sites[k,c('downstream_NA')] <- nrow(Down_streams[is.na(Down_streams$MMI),])
      sites[k,c('downstream_large_dams')] <- nrow(Down_dams[Down_dams$up_weight==.25,])
      sites[k,c('downstream_small_dams')] <- nrow(Down_dams[Down_dams$up_weight==0.8,])
      sites[k,c('downstream_canals')] <- nrow(Down_canals)
      sites[k,c('downstream_large_lakes')] <- nrow(Down_wb_swamp[Down_wb_swamp$Desc %in% c('Lake/Pond Perennial','Lake/Pond Intermittent') & Down_wb_swamp$AreaSqKm >= 19,])
      sites[k,c('downstream_small_lakes')] <- nrow(Down_wb_swamp[Down_wb_swamp$Desc %in% c('Lake/Pond Perennial','Lake/Pond Intermittent') & Down_wb_swamp$AreaSqKm < 2,])
      sites[k,c('downstream_swamps')] <- nrow(Down_wb_swamp[Down_wb_swamp$Desc %in% c("Swamp/Marsh","Swamp/Marsh Perennial","Swamp/Marsh Intermittent"),])
      sites[k,c('downstream_crossings')] <- nrow(Down_crossings)
      sites[k,c('downstream_flowlength')] <- as.numeric(units::set_units(sum(st_length(Down_streams)), km))
      sites[k,c('downstream_mean_MMI')] <- mean(Down_streams$MMI,na.rm = T)
    }
