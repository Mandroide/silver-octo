# silver-octo

```java
// src/main/java/com/example/spec/Specs.java
package com.example.spec;

import org.springframework.data.jpa.domain.Specification;

import jakarta.persistence.criteria.*;
import jakarta.persistence.metamodel.*;
import java.util.*;
import java.util.function.Function;

public final class Specs {

  private Specs() {}

  /* ---------- Base combinators (no where(..)) ---------- */

  public static <T> Specification<T> alwaysTrue() {
    return (root, q, cb) -> cb.conjunction();
  }

  public static <T> Specification<T> alwaysFalse() {
    return (root, q, cb) -> cb.disjunction();
  }

  /** AND all non-null specs. If empty → true. */
  @SafeVarargs
  public static <T> Specification<T> andAll(Specification<T>... specs) {
    return andAll(Arrays.asList(specs));
  }

  public static <T> Specification<T> andAll(Collection<Specification<T>> specs) {
    return specs.stream().filter(Objects::nonNull)
        .reduce(alwaysTrue(), Specification::and);
  }

  /** OR any non-null specs. If empty → false. */
  @SafeVarargs
  public static <T> Specification<T> orAny(Specification<T>... specs) {
    return orAny(Arrays.asList(specs));
  }

  public static <T> Specification<T> orAny(Collection<Specification<T>> specs) {
    return specs.stream().filter(Objects::nonNull)
        .reduce(alwaysFalse(), Specification::or);
  }

  /* ---------- Simple attributes ---------- */

  public static <T, Y> Specification<T> eq(SingularAttribute<? super T, Y> attr, Y value) {
    return (root, q, cb) -> value == null ? null : cb.equal(root.get(attr), value);
  }

  public static <T, Y> Specification<T> ne(SingularAttribute<? super T, Y> attr, Y value) {
    return (root, q, cb) -> value == null ? null : cb.notEqual(root.get(attr), value);
  }

  public static <T> Specification<T> like(SingularAttribute<? super T, String> attr, String pattern) {
    return (root, q, cb) -> (pattern == null || pattern.isBlank()) ? null
        : cb.like(root.get(attr), pattern);
  }

  /** case-insensitive contains */
  public static <T> Specification<T> ilike(SingularAttribute<? super T, String> attr, String text) {
    return (root, q, cb) -> (text == null || text.isBlank()) ? null
        : cb.like(cb.lower(root.get(attr)), "%" + text.toLowerCase(Locale.ROOT) + "%");
  }

  public static <T, Y extends Comparable<? super Y>> Specification<T> gt(SingularAttribute<? super T, Y> attr, Y value) {
    return (root, q, cb) -> value == null ? null : cb.greaterThan(root.get(attr), value);
  }

  public static <T, Y extends Comparable<? super Y>> Specification<T> ge(SingularAttribute<? super T, Y> attr, Y value) {
    return (root, q, cb) -> value == null ? null : cb.greaterThanOrEqualTo(root.get(attr), value);
  }

  public static <T, Y extends Comparable<? super Y>> Specification<T> lt(SingularAttribute<? super T, Y> attr, Y value) {
    return (root, q, cb) -> value == null ? null : cb.lessThan(root.get(attr), value);
  }

  public static <T, Y extends Comparable<? super Y>> Specification<T> le(SingularAttribute<? super T, Y> attr, Y value) {
    return (root, q, cb) -> value == null ? null : cb.lessThanOrEqualTo(root.get(attr), value);
  }

  public static <T, Y extends Comparable<? super Y>> Specification<T> between(
      SingularAttribute<? super T, Y> attr, Y from, Y to) {
    return (root, q, cb) -> (from == null || to == null) ? null : cb.between(root.get(attr), from, to);
  }

  public static <T, Y> Specification<T> in(SingularAttribute<? super T, Y> attr, Collection<Y> values) {
    return (root, q, cb) -> {
      if (values == null || values.isEmpty()) return null;
      CriteriaBuilder.In<Y> in = cb.in(root.get(attr));
      values.forEach(in::value);
      return in;
    };
  }

  public static <T, Y> Specification<T> isNull(SingularAttribute<? super T, Y> attr) {
    return (root, q, cb) -> cb.isNull(root.get(attr));
  }

  public static <T, Y> Specification<T> isNotNull(SingularAttribute<? super T, Y> attr) {
    return (root, q, cb) -> cb.isNotNull(root.get(attr));
  }

  /* ---------- Nested (singular) paths: entityAttr.subAttr ---------- */

  public static <T, R, Y> Specification<T> eqNested(
      SingularAttribute<? super T, R> entityAttr,
      SingularAttribute<? super R, Y> nestedAttr,
      Y value) {
    return (root, q, cb) -> value == null ? null : cb.equal(root.get(entityAttr).get(nestedAttr), value);
  }

  public static <T, R> Specification<T> ilikeNested(
      SingularAttribute<? super T, R> entityAttr,
      SingularAttribute<? super R, String> nestedAttr,
      String text) {
    return (root, q, cb) -> (text == null || text.isBlank()) ? null
        : cb.like(cb.lower(root.get(entityAttr).get(nestedAttr)), "%" + text.toLowerCase(Locale.ROOT) + "%");
  }

  /* ---------- Collection joins (OneToMany/ManyToMany) ---------- */

  public static <T, R, Y> Specification<T> joinEq(
      SetAttribute<? super T, R> collectionAttr,
      SingularAttribute<? super R, Y> joinedAttr,
      Y value) {
    return (root, q, cb) -> {
      if (value == null) return null;
      // Avoid duplicates when joining
      q.distinct(true);
      Join<T, R> join = root.join(collectionAttr, JoinType.INNER);
      return cb.equal(join.get(joinedAttr), value);
    };
  }

  public static <T, R> Specification<T> joinIlike(
      SetAttribute<? super T, R> collectionAttr,
      SingularAttribute<? super R, String> joinedAttr,
      String text) {
    return (root, q, cb) -> {
      if (text == null || text.isBlank()) return null;
      q.distinct(true);
      Join<T, R> join = root.join(collectionAttr, JoinType.INNER);
      return cb.like(cb.lower(join.get(joinedAttr)), "%" + text.toLowerCase(Locale.ROOT) + "%");
    };
  }

  public static <T, R, Y> Specification<T> joinIn(
      SetAttribute<? super T, R> collectionAttr,
      SingularAttribute<? super R, Y> joinedAttr,
      Collection<Y> values) {
    return (root, q, cb) -> {
      if (values == null || values.isEmpty()) return null;
      q.distinct(true);
      Join<T, R> join = root.join(collectionAttr, JoinType.INNER);
      CriteriaBuilder.In<Y> in = cb.in(join.get(joinedAttr));
      values.forEach(in::value);
      return in;
    };
  }

  /* ---------- Power users: custom path function ---------- */

  public static <T, Y> Specification<T> custom(Function<Root<T>, Path<Y>> path,
                                               Function<CriteriaBuilder, javax.persistence.criteria.Expression<Boolean>> predicateBuilder) {
    // Intentionally left flexible for advanced use-cases
    return (root, q, cb) -> predicateBuilder.apply(cb);
  }
}

```
