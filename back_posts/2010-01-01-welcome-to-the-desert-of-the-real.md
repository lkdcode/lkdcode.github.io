---
date: 2019-05-16 23:48:05
layout: post
title: Welcome to the desert of the real
subtitle: 'Lorem ipsum dolor sit amet, consectetur adipisicing elit.'
description: >-
  Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
  tempor incididunt ut labore et dolore magna aliqua.
image: >-
  https://res.cloudinary.com/dm7h7e8xj/image/upload/v1559821647/theme6_qeeojf.jpg
optimized_image: >-
  https://res.cloudinary.com/dm7h7e8xj/image/upload/c_scale,w_380/v1559821647/theme6_qeeojf.jpg
category: blog
tags:
  - welcome
  - blog
author: mranderson
paginate: true
---
Cas sociis natoque penatibus et magnis <a href="#">dis parturient montes</a>, nascetur ridiculus mus. *Aenean eu leo quam.* Pellentesque ornare sem lacinia quam venenatis vestibulum. Sed posuere consectetur est at lobortis. Cras mattis consectetur purus sit amet fermentum.

> Curabitur blandit tempus porttitor. Nullam quis risus eget urna mollis ornare vel eu leo. Nullam id dolor id nibh ultricies vehicula ut id elit.

Etiam porta **sem malesuada magna** mollis euismod. Cras mattis consectetur purus sit amet fermentum. Aenean lacinia bibendum nulla sed consectetur.

## Inline HTML elements

HTML defines a long list of available inline tags, a complete list of which can be found on the [Mozilla Developer Network](https://developer.mozilla.org/en-US/docs/Web/HTML/Element).

* **To bold text**, use `<strong>`.
* *To italicize text*, use `<em>`.
* Abbreviations, like <abbr title="HyperText Markup Langage">HTML</abbr> should use `<abbr>`, with an optional `title` attribute for the full phrase.
* Citations, like <cite>&mdash; Thomas A. Anderson</cite>, should use `<cite>`.
* <del>Deleted</del> text should use `<del>` and <ins>inserted</ins> text should use `<ins>`.
* Superscript <sup>text</sup> uses `<sup>` and subscript <sub>text</sub> uses `<sub>`.

Most of these elements are styled by browsers with few modifications on our part.

# Heading 1

## Heading 2

### Heading 3

#### Heading 4

Vivamus sagittis lacus vel augue rutrum faucibus dolor auctor. Duis mollis, est non commodo luctus, nisi erat porttitor ligula, eget lacinia odio sem nec elit. Morbi leo risus, porta ac consectetur ac, vestibulum at eros.

--page-break--

## Code

Cum sociis natoque penatibus et magnis dis `code element` montes, nascetur ridiculus mus.

```js
// Example can be run directly in your JavaScript console

// Create a function that takes two arguments and returns the sum of those arguments
var adder = new Function("a", "b", "return a + b");

// Call the function
adder(2, 6);
// > 8
```

자바를 한 번 써볼게용

```java
package hotil.baemo.domains.clubs.adapter.input.rest.query;

import hotil.baemo.core.common.response.ResponseDTO;
import hotil.baemo.core.security.oauth2.persistence.entity.BaeMoOAuth2User;
import hotil.baemo.domains.clubs.application.dto.query.ClubsResponse;
import hotil.baemo.domains.clubs.application.usecases.query.RetrieveHomeUseCase;
import hotil.baemo.domains.clubs.domain.entity.clubs.ClubsId;
import hotil.baemo.domains.clubs.domain.entity.clubs.ClubsUserId;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequiredArgsConstructor
@RequestMapping("/api/clubs")
public class QueryClubsApi {
    private final RetrieveHomeUseCase retrieveHomeUseCase;

    @GetMapping("{clubsId}")
    public ResponseDTO<ClubsResponse.HomeDTO> getHome(
            @AuthenticationPrincipal BaeMoOAuth2User user,
            @PathVariable(name = "clubsId") final Long clubsId
    ) {
        final var response = retrieveHomeUseCase.retrieve(
                new ClubsUserId(user.getId()),
                new ClubsId(clubsId));

        return ResponseDTO.ok(response);
    }
}
```


```java
package hotil.baemo.domains.clubs.domain.entity.clubs;

import hotil.baemo.core.common.response.ResponseCode;
import hotil.baemo.core.common.response.exception.CustomException;
import hotil.baemo.domains.clubs.domain.entity.post.ClubsPost;
import hotil.baemo.domains.clubs.domain.entity.post.ClubsPostWriter;
import hotil.baemo.domains.clubs.domain.specification.command.ClubsPostSpecification;
import hotil.baemo.domains.clubs.domain.specification.command.ClubsSpecification;
import hotil.baemo.domains.clubs.domain.value.clubs.ClubsRole;
import hotil.baemo.domains.clubs.domain.value.clubs.Join;
import lombok.Builder;
import lombok.Getter;

import java.util.Objects;

@Getter
public class ClubsAdmin implements ClubsUser {
    private final ClubsRole role = ClubsRole.ADMIN;
    private final ClubsUserId clubsUserId;
    private final ClubsId clubsId;

    @Builder
    public ClubsAdmin(ClubsUserId clubsUserId, ClubsId clubsId) {
        this.clubsUserId = clubsUserId;
        this.clubsId = clubsId;
    }

    @Override
    public ClubsPostSpecification.Create createPostSpec() {
        return ((title, content, images) -> ClubsPost.builder()
                .clubsPostWriter(new ClubsPostWriter(this.clubsUserId.id()))
                .clubsPostTitle(title)
                .clubsPostContent(content)
                .clubsPostImages(images)
                .build());
    }

    @Override
    public ClubsPostSpecification.Update updatePostSpec() {
        return ((clubsPost, newTitle, newContent, newImages) -> {
            if (!Objects.equals(clubsPost.getClubsPostId().id(), this.clubsUserId.id())) {
                throw new CustomException(ResponseCode.AUTH_RESTRICTED);
            }
            clubsPost.updateTitle(newTitle);
            clubsPost.updateContent(newContent);
            clubsPost.updateImages(newImages);
            return clubsPost;
        });
    }

    @Override
    public ClubsPostSpecification.Delete deletePostSpec() {
        return ClubsPost::delete;
    }

    @Override
    public ClubsSpecification.Update updateClubs() {
        return (oldClubs, newClubs) -> {
            oldClubs.updateClubsName(newClubs.getClubsName());
            oldClubs.updateClubsSimpleDescription(newClubs.getClubsSimpleDescription());
            oldClubs.updateClubsDescription(newClubs.getClubsDescription());
            oldClubs.updateClubsLocation(newClubs.getClubsLocation());
//            oldClubs.updateClubsProfileImage(newClubs.getClubsProfileImage());
//            oldClubs.updateClubsBackgroundImage(newClubs.getClubsBackgroundImage());
            // TODO : MultipartFile vs String 문제 해결을 해야 한다.
            return oldClubs;
        };
    }

    @Override
    public ClubsSpecification.Delete deleteClubs() {
        return clubs -> {
            if (!Objects.equals(clubs.getClubsAdmin().clubsUserId.id(), this.clubsUserId.id())) {
                throw new CustomException(ResponseCode.AUTH_RESTRICTED);
            }
            return clubs.delete();
        };
    }

    @Override
    public Long getClubsId() {
        return this.clubsId.clubsId();
    }

    @Override
    public Long getClubsUserId() {
        return this.clubsUserId.id();
    }

    @Override
    public ExecuteJoin executeJoin() {
        throw new CustomException(ResponseCode.CLUBS_ALREADY_MEMBER);
    }

    @Override
    public ExecuteHandle executeHandle() {
        return (clubsNonMember, isAccept) -> Join.Result.builder()
                .clubsNonMember(clubsNonMember)
                .isAccept(isAccept)
                .build();
    }
}
```

너무 너무 구린디,...우짜지

Aenean lacinia bibendum nulla sed consectetur. Etiam porta sem malesuada magna mollis euismod. Fusce dapibus, tellus ac cursus commodo, tortor mauris condimentum nibh, ut fermentum massa.

## Lists

Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Aenean lacinia bibendum nulla sed consectetur. Etiam porta sem malesuada magna mollis euismod. Fusce dapibus, tellus ac cursus commodo, tortor mauris condimentum nibh, ut fermentum massa justo sit amet risus.

* Praesent commodo cursus magna, vel scelerisque nisl consectetur et.
* Donec id elit non mi porta gravida at eget metus.
* Nulla vitae elit libero, a pharetra augue.

Donec ullamcorper nulla non metus auctor fringilla. Nulla vitae elit libero, a pharetra augue.

1. Vestibulum id ligula porta felis euismod semper.
2. Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus.
3. Maecenas sed diam eget risus varius blandit sit amet non magna.

Cras mattis consectetur purus sit amet fermentum. Sed posuere consectetur est at lobortis.

Integer posuere erat a ante venenatis dapibus posuere velit aliquet. Morbi leo risus, porta ac consectetur ac, vestibulum at eros. Nullam quis risus eget urna mollis ornare vel eu leo.

## Images

Quisque consequat sapien eget quam rhoncus, sit amet laoreet diam tempus. Aliquam aliquam metus erat, a pulvinar turpis suscipit at.

![placeholder](https://placehold.it/800x400 "Large example image") ![placeholder](https://placehold.it/400x200 "Medium example image") ![placeholder](https://placehold.it/200x200 "Small example image")

[ggggg](https://github.com/JavaDocument/spring-kafka-redis/blob/main/docs/image/skr1.png "test")

## Tables

Aenean lacinia bibendum nulla sed consectetur. Lorem ipsum dolor sit amet, consectetur adipiscing elit.

<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Upvotes</th>
      <th>Downvotes</th>
    </tr>
  </thead>
  <tfoot>
    <tr>
      <td>Totals</td>
      <td>21</td>
      <td>23</td>
    </tr>
  </tfoot>
  <tbody>
    <tr>
      <td>Alice</td>
      <td>10</td>
      <td>11</td>
    </tr>
    <tr>
      <td>Bob</td>
      <td>4</td>
      <td>3</td>
    </tr>
    <tr>
      <td>Charlie</td>
      <td>7</td>
      <td>9</td>
    </tr>
  </tbody>
</table>

Nullam id dolor id nibh ultricies vehicula ut id elit. Sed posuere consectetur est at lobortis. Nullam quis risus eget urna mollis ornare vel eu leo.
