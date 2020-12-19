# Erratas del Libro Postgis:Análisis Espacial Avanzado
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
