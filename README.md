# Erratas del Libro Postgis:Análisis Espacial Avanzado
Repositorio para registrar las erratas del libro Postgis: Análisis Espacial Avanzado. Segunda edición.

* Página 21: Vistas como control dínamico de la calidad cartográfica. La query hacer referencia a la tabla psuelos (la cual no tiene ninguna superposición) en lugar de la tabla suelos. Además en el cast de la geometría el SRID está equivocado debiendo ser 23030.

```sql
create or replace view ej1.mustnotoverlap as 
	  select s1.gid as gid, s1.geom::geometry(MULTIPOLYGON,23030) as geom 
	  from psuelos s1, psuelos s2 
	  where (st_overlaps (s1.geom, s2.geom) 
		  or st_covers (s1.geom, s2.geom) 
		  or st_covers (s2.geom, s1.geom)) and s1.gid <> s2.gid;
```
