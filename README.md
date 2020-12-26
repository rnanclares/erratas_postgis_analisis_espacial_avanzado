# Erratas del Libro Postgis: Análisis Espacial Avanzado
Repositorio para registrar las erratas del libro Postgis: Análisis Espacial Avanzado. Segunda edición. La revisión se ha hecho usando Postgresql 13.1 y Postgis 3.0.3. 

* Página 121: **Vistas como control dínamico de la calidad cartográfica**. La query hacer referencia a la tabla psuelos (la cual no tiene ninguna superposición) en lugar de la tabla suelos, ocurre en el libro y en el archivo `capb.sql`. Además en el cast de la geometría el SRID está equivocado debiendo ser 23030.

```sql
create or replace view ej1.mustnotoverlap as 
	  select s1.gid as gid, s1.geom::geometry(MULTIPOLYGON,23030) as geom 
	  from suelos s1, suelos s2 
	  where (st_overlaps (s1.geom, s2.geom) 
		  or st_covers (s1.geom, s2.geom) 
		  or st_covers (s2.geom, s1.geom)) and s1.gid <> s2.gid;
```

* Página 133: **Utilización de STX_EXTRACT**. La query hace referencia a la tabla suelos la cual contiene un error topológico (ERROR:  lwgeom_intersection: GEOS Error: TopologyException: Input geom 0 is invalid: Ring Self-intersection at or near point 730693.1875 4686105 at 730693.1875 4686105) lo cual impide la ejecución de la misma. Se puede evitar usando la función st_makevalid sobre la geometría de la tabla suelos.

```sql
select geometrytype(geom) as tipo, count(*) from
(Select st_intersection(st_makevalid(s.geom), t.geom) as geom from ttmm t, suelos s
where st_intersects(st_makevalid(s.geom), t.geom)) as tabla group by tipo;
```
o arreglamos la geometría de la tabla

```sql
update suelos
set geom = st_makevalid(geom)::geometry(multipolygon, 23030);
```
* Página 139: *Borrado en un solo paso* La query propuesta en este apartado está incompleta, el resultado de la query no contiene los polígonos que no cumplen la condición de st_relate('T&ast;&ast;&ast;&ast;&ast;&ast;&ast;&ast;').

```sql
insert into erase1b (tema, grupo, geom)
select tema, grupo, geom 
from
(select tema, grupo, count(n.gid) as numright, s.geom as geom_completo,
 stx_extract(st_difference(s.geom,
 coalesce(st_union(n.geom), 'GEOMETRYCOLLECTION EMPTY'::geometry(geometry, 23030))), 2) as geom
 from suelos s left join nucleos n
 on s.geom && n.geom and st_relate(s.geom, n.geom, 'T********') group by s.gid
) as tabla where (numright > 0 and geom is not null) or numright = 0;
```

Sin usar stx_extract()

```sql
insert into erase1b (tema, grupo, geom)
select tema, grupo, geom 
from
(select tema, grupo, count(n.gid) as numright, s.geom as geom_completo,
 st_multi(st_collectionextract(st_difference(s.geom,
	coalesce(st_union(n.geom), 'GEOMETRYCOLLECTION EMPTY'::geometry(geometry, 23030))), 3)) as geom
 from suelos s left join nucleos n
 on s.geom && n.geom and st_relate(s.geom, n.geom, 'T********') group by s.gid
) as tabla where (numright > 0 and geom is not null) or numright = 0;
```
* Página 140 - Superposición(Overlay) - Ambas diferencias están incompletas

```sql
create table superpos1 (gid serial primary key, ine varchar, tema varchar, grupo varchar,
						geom geometry(multipolygon, 23030));
					   
-- Diferencia (Suelos - Nucleos)
insert into superpos1 (tema, grupo, geom)
select tema, grupo, geom
from
(select tema, grupo, count(n.gid) as numright,
 stx_extract(st_difference(s.geom, coalesce(st_union(n.geom), 'GEOMETRYCOLLECTION EMPTY'::geometry(geometry, 23030))), 2) as geom
 from suelos s left join nucleos n on s.geom && n.geom and st_relate(s.geom, n.geom, 'T********') group by s.gid
) as tabla where (numright > 0 and geom is not null) or numright = 0;

-- Intersección Suelos Nucleos
insert into superpos1 (ine, tema, grupo, geom)
select n.ine, s.tema, s.grupo, stx_extract(st_intersection(n.geom, s.geom), 2)
from nucleos n, suelos s
where n.geom && s.geom and st_relate(n.geom, s.geom, 'T********');

-- Diferencia (Nucleos - Suelos)

insert into superpos1 (ine, geom)
select ine, geom
from
(select ine, count(n.gid) as numright,
 stx_extract(st_difference(n.geom, coalesce(st_union(s.geom), 'GEOMETRYCOLLECTION EMPTY'::geometry(geometry, 23030))), 2) as geom
 from nucleos n left join suelos s on n.geom && s.geom and st_relate(n.geom, s.geom, 'T********') group by n.gid
) as tabla where (numright > 0 and geom is not null) or numright = 0;
```

* Página 144 - Recorte en dos fases. La geometría está definidia como MULTIPOLYGON cuando obviamente el resultado del recorte es una geometría de tipo MULTILINESTRING.

```sql
drop table if exists viaria1 cascade;
create table viaria1 (gid serial primary key, tipo integer, geom geometry(multilinestring, 23030));
insert into viaria1 (tipo, geom)
select v.tipo, stx_extract(st_intersection(v.geom, t.geom), 1) as geom
from viariache v, ttmmdis t
where v.geom && t.geom and st_relate(v.geom, t.geom, 'T********');
```
